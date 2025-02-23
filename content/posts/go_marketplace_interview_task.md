+++
date = '2025-02-21T17:20:48+07:00'
title = 'Разбор задачи с собеседования на должность senior Go разработчика в крупный маркетплейс'
showTags = false
hideBackToTop = true
+++

## Предыстория

Смотрел запись собеседования на должность senior Go разработчика в крупный маркетплейс на youtube.com 
Увидел задачу, решил разобрать два варианта её решения.

Для понимания того, что будет происходить далее, вы должны быть знакомы с основами языка программирования Go.  
Самый простой способ пощупать Go -- это пройти интерактивный [Go tour](https://go.dev/tour/list) от создателей языка.

## Условие задачи

Дан код:
```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	start := time.Now()
	m, err := Do(context.Background(), []User{{"John"}, {"Bob"}, {"Alice"}, {"Richard"}})
	fmt.Println("time:", time.Since(start))
	fmt.Println(m, err)
}

type User struct {
	Name string
}

func fetch(ctx context.Context, u User) (string, error) {
	time.Sleep(time.Millisecond * 10)
	return u.Name, nil
}

func Do(ctx context.Context, users []User) (map[string]int64, error) {
	names := make(map[string]int64, 0)
	for _, u := range users {
		name, err := fetch(ctx, u)
		if err != nil {
			return nil, err
		}
		names[name] += 1
	}
	return names, nil
}
```

Код отрабатывает приблизительно за 40ms.  

Необходимо:
- Доработать функцию `Do` таким образом, чтобы код отрабатывал приблизительно за 10ms.   
- При первой ошибке функция `Do` должна завершить свое исполнение и вернуть ошибку. 

Код функции `fetch` редактировать нельзя.

## Решение 1

Идея: выполнять блокирующие операции в отдельных горутинах и останавливать их выполнение при возникновении первой ошибки.

Блокирующей операцией является вызов `fetch` внутри цикла в `Do`, соответственно вызовы `fetch` нужно делать в отдельных горутинах внутри цикла по `users` в `Do`. Дождаться выполнения всех горутин в `Do` можно при помощи `sync.WaitGroup`. Т.к. доступ к `names` будет выполняться из разных горутин, то для контроля доступа к этой переменной нужно добавить `sync.Mutex`.

В случае возврата ошибки от `fetch` необходимо:
1. Прокинуть ошибку из горутины в функцию `Do`. Нужно это для того, чтобы вернуть полученную ошибку из функции `Do`.
2. Завершить уже запущенные вызовы `fetch`  и прекратить запуск новых вызовов `fetch`.

Для прокидывания ошибки из горутины в `Do` можно использовать запись в буферизированный канал, который будет объявлен в `Do`. При этом размер буфера канала должен быть равен количеству запускаемых горутин в цикле по `users`, т.к. при таком размере буфера все запущенные горутины смогут записать возможные ошибки без блокировок.

Для остановки запущенных вызовов `fetch` и прекращения запуска новых вызовов `fetch` в горутинах следует использовать [контекст с возможностью отмены](https://go.dev/doc/database/cancel-operations), который будет создаваться из `ctx`, приходящего в `Do`. При получении первой ошибки в горутине с выполняемым `fetch` контекст будет отменен при помощи вызова соответствующей функции `cancel`. Перед вызовом `fetch` в горутинах нужно будет проверить, что контекст не был отменен.

В `Do` после запуска всех горутин нужно будет дождаться их выполнения. Затем закрыть канал для записи ошибок (т.к. все пишущие горутины завершились и больше писать в канал никто не будет, проверить есть ли в этой канале хотя бы одна ошибка и вернуть результат.

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

func main() {
	start := time.Now()
	m, err := Do(context.Background(), []User{{"John"}, {"Bob"}, {"Alice"}, {"Richard"}})
	fmt.Println("time:", time.Since(start))
	fmt.Println(m, err)
}

type User struct {
	Name string
}

// В реальном мире функция делала бы сетевой вызов и использовала бы context для сигнала об отмене операции.
func fetch(_ context.Context, u User) (string, error) {
	time.Sleep(time.Millisecond * 10)
	//return "", fmt.Errorf("some fetch error")
	return u.Name, nil
}

func Do(ctx context.Context, users []User) (map[string]int64, error) {
	n := len(users)
	// Создать канал для записи ошибок. Размер буфера - количество запускаемых горутин.
	errCh := make(chan error, n)
	// Создать контекст с возможностью отмены.
	ctx2, cancel := context.WithCancel(ctx)
	// Завершить контекст для случая, если не произошло ни одной ошибки.
    // Повторный вызов cancel ни на что не повлияет. 
	defer cancel()
	// Создать мьютекс для управления доступом к names
	var mx sync.Mutex
	names := make(map[string]int64)
	// Создать группу ожидания для ожидания выполнения всех горутин, внутри которых будет вызываться fetch
	var wg sync.WaitGroup
	wg.Add(n)
	for _, u := range users {
		// Не передаем переменную u в параметрах функций, т.к. начиная с go1.22 переменная u создается на каждой итерации цикла.
		go func() {
			defer wg.Done()
			select {
			// Контекст был завершен из-за ошибки, прекратить выполнение.
			case <-ctx2.Done():
				return
			// Ошибки пока что не было, контекст не отменен, продолжаем выполнение
			default:
				name, err := fetch(ctx2, u)
				if err != nil {
					// Отменить контекст и отменить выполнение запущенных горутин.
					cancel()
					// Записать ошибку для чтения в Do.
					errCh <- err
					return
				}
				mx.Lock()
				names[name]++
				mx.Unlock()
			}
		}()
	}
	// Ожидать выполнение всех запущенных горутин.
	wg.Wait()
	// Закрыть канал, т.к. все писатели отработали и больше в него писать никто не будет.
	close(errCh)
	// После закрытия буферизированного канала из него можно читать, будет вычитано содержимое его буфера (если оно есть).
	for err := range errCh {
		// Вернуть первую вычитанную ошибку.
		return nil, err
	}
	// Канал errCh оказался пустым, ошибок не было, вернуть результат работы
	return names, nil
}
```

## Решение 2

Сделать тоже самое, что и в решении 1, используя [errgroup](https://pkg.go.dev/golang.org/x/sync/errgroup).

`errgroup.Group` позволяет запускать выполнение функций в отдельных горутинах при помощи метода `Go(func() error)`, дожидаться их выполнения при помощи метода `Wait()`. 

`errgroup.Group` автоматически останавливает запуск новых горутин как только одна из уже запущенных горутин вернула не nil ошибку. 

Так же `errgroup.Group` позволяет ограничивать количество одновременно выполняемых горутин при помощи метода `SetLimit()`.

`errgroup.WithContext()` помимо `errgroup.Group` вернет контекст, который будет отменен как только одна из уже запущенных горутин вернула не nil ошибку.

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"

	"golang.org/x/sync/errgroup"
)

func main() {
	start := time.Now()
	m, err := Do(context.Background(), []User{{"John"}, {"Bob"}, {"Alice"}, {"Richard"}})
	fmt.Println("time:", time.Since(start))
	fmt.Println(m, err)
}

type User struct {
	Name string
}

// В реальном мире функция делала бы сетевой вызов и использовала бы context для сигнала об отмене операции.
func fetch(_ context.Context, u User) (string, error) {
	time.Sleep(time.Millisecond * 10)
	//return "", fmt.Errorf("some fetch error")
	return u.Name, nil
}

func Do(ctx context.Context, users []User) (map[string]int64, error) {
	var mx sync.Mutex
	names := make(map[string]int64)
	// Создать errGroup для запускать горутин.
	// ctx2 автоматически отменится после получения первой не nil ошибки внутри g
	g, ctx2 := errgroup.WithContext(ctx)
	for _, u := range users {
		// Запустить горутины для вызова fetch
		g.Go(func() error {
			name, err := fetch(ctx2, u)
			if err != nil {
				return err
			}
			mx.Lock()
			names[name]++
			mx.Unlock()
			return nil
		})
	}
	// Дождаться выполнения
	if err := g.Wait(); err != nil {
		// Если возникла ошибка, то вернут её
		return nil, err
	}
	// Вернуть результат
	return names, nil
}

```

## Ссылки на дополнительные материалы

-  [Go tour](https://go.dev/tour/list)
-  [Err group](https://pkg.go.dev/golang.org/x/sync/errgroup)
-  [Context](https://go.dev/blog/context)
-  [Fixing For Loops in Go 1.22](https://go.dev/blog/loopvar-preview)
-  [Mutexes](https://gobyexample.com/mutexes)
-  [Goroutines](https://gobyexample.com/goroutines)
-  [Buffered channel ](https://gobyexample.com/channel-buffering)
