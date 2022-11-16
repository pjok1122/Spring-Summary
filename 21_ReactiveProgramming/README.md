# PubSub / Observer

iterator는 pull 방식
observer pattern은 push 방식
- observable(Publisher) -> DATA/Event -> observer(subscriber)
- 비동기적으로 asynchronous하게 처리할 수 있음.


1. 기존의 Java Observerable에는 끝이라는 개념이 없음. (더 이상 데이터가 없다 라는 의미)
시간 제한이 있다거나, DB에서 데이터를 긁어오는 연산이 끝났을 때 종료할 수 있어야 하지 않을까? 무한정 동작하기 때문 -> complete가 필요하다.

2. 에러에 대한 처리를 패턴에 녹이지 않았음. Observer에서 각자 구현함.

이 두 가지를 추가해서 새로운 Observer pattern을 구현. 이게 Reactive Programming의 한 축

java의 reactive programming은 `www.reactive-streams.org` 에 바탕을 두고 설계되었음.


Publisher <-> Subscriber 통신 프로토콜

1) subscriber : 구독요청. `publisher.subscribe(subscriber)`
2) publisher : onSubscribe 호출. 해당 메서드에는 `subscription`에 대한 요구사항이 쓰여있음. `subscriber.onSubscribe(subscription)`
3) publisher : onNext 호출. 데이터가 있을 경우, subscriber에게 데이터 전달.
4) publisher : subscriber가 subscription에 요청한 데이터의 개수만큼 onNext를 반복 수행. 
5-1) publisher : 더 이상 읽을 데이터가 없을 경우 onComplete 호출
5-2) publisher : 처리 중 에러가 발생하는 경우 onError 호출


```java
public class PubSub {
    public static void main(String[] args) throws InterruptedException {
        Iterable<Integer> iter = Arrays.asList(1, 2, 3, 4, 5);
        ExecutorService es = Executors.newSingleThreadExecutor();
        // 구독하는 방법에 대한 기준만 정의.
        Publisher p = new Publisher() {
            @Override
            public void subscribe(Subscriber subscriber) {
                Iterator<Integer> it = iter.iterator();     //DB에서 조회한 데이터라고 생각.

                subscriber.onSubscribe(new Subscription() {
                    @Override
                    public void request(long n) {

                        es.execute(() -> {
                            int i = 0;
                            try {
                                while (i++ < n) {
                                    if (it.hasNext()) {
                                        subscriber.onNext(it.next());
                                    } else {
                                        subscriber.onComplete();
                                        break;
                                    }
                                }
                            } catch (RuntimeException e) {
                                subscriber.onError(e);
                            }
                        });
                    }

                    @Override
                    public void cancel() {

                    }
                });
            }
        };

        Subscriber<Integer> sub = new Subscriber<>() {
            Subscription subscription;

            @Override
            public void onSubscribe(Subscription subscription) {
                // 구독이라는 중간 매개체를 전달 받음. (처리 속도를 조절할 수 있음)
                // Publisher -> [Subscription] -> Subscriber
                System.out.println(Thread.currentThread().getName() + " onSubscribe");
                this.subscription = subscription;
                this.subscription.request(2L);                 //subscriber가 처리할 수 있는 데이터의 양을 요구.
                                                                 // request 즉시 데이터를 달라는 의미는 아니고 구독에 대한 상세 설정 같은 느낌. 따라서 pull 방식과 다름 (pub/sub의 속도 차 조절도 가능해짐).
                                                                 // 여기서 속도 조절이 가능함.

            }

            int bufferSize = 2;

            @Override
            public void onNext(Integer item) {  //publisher가 데이터를 주면 onNext에서 받음.
                System.out.println(Thread.currentThread().getName() + " onNext - " + item);

                //처리가 끝났으니 publisher에게 '구독'을 통해서 현재 몇 개 처리할 수 있는지 상황을 알려줌. (버퍼의 사이즈가 절반찼을 때 받는다거나 등등)
                if (--bufferSize <= 0) {
                    bufferSize = 2;
                    this.subscription.request(2);
                }
            }

            @Override
            public void onError(Throwable throwable) {      //에러가 발생하면 해당 메서드가 받음. 따라서 try-catch 구문이 필요가 없음. 이 메서드가 호출되면 더 이상의 구독은 종료.
                System.out.println("onError: " + throwable.getMessage());
            }

            @Override
            public void onComplete() {      //publisher가 더 이상 처리할 수 있는 데이터가 없을 때 subscriber의 onComplete 메서드를 호출해줄 수 있음. 이 메서드가 호출되면 더 이상의 구독은 종료.
                System.out.println(Thread.currentThread().getName() + " onComplete");
            }
        };

        p.subscribe(sub);

        es.awaitTermination(10, TimeUnit.SECONDS);
        es.shutdown();
    }
}
```


## Operators

Publisher -> [Data1] -> Operator1 -> [Data2] -> Operator2 -> [Data3] -> Subscriber

publisher와 subscriber 사이에서 데이터를 가공하는 주체를 Operator라고 함.

publisher -> [Data1] -> mapPublisher -> [Data2] -> Subscriber
publisher -> [Data1] -> reducePublisher -> [Data2] -> Subscriber

중간에 있는 operator는 subscriber의 구독 요청을 publisher에게 넘겨주고, onNext에서 subscriber에게 데이터를 어떻게 넘겨줄지 가공을 해서 넘기게 됨.

```java
public class PubSub {
    public static void main(String[] args) {
        Publisher<Integer> pub = iterPub(Stream.iterate(1, a -> a + 1).limit(10).collect(Collectors.toList()));
        Publisher<String> mapPub = mapPub(pub, s -> "[" + s * 10 + "]");
//        Publisher<Integer> sumPub = sumPub(mapPub);
        Publisher<StringBuilder> reducePub = reduceMap(pub, new StringBuilder(),
                                                       (a, b) -> a.append(b+","));
        reducePub.subscribe(logSub());
    }

    private static <T,R> Publisher<R> reduceMap(Publisher<T> pub, R init,
                                                BiFunction<R, T, R> bf) {
        return new Publisher<R>() {
            @Override
            public void subscribe(Subscriber<? super R> sub) {
                pub.subscribe(new DelegateSub<T, R>(sub) {
                    R result = init;

                    @Override
                    public void onNext(T i) {
                        result = bf.apply(result, i);
                    }

                    @Override
                    public void onComplete() {
                        sub.onNext(result);
                        sub.onComplete();
                    }
                });
            }
        };
    }

    private static <T,R> Publisher<R> mapPub(Publisher<T> pub,
                                             Function<T, R> f) {
        return new Publisher<R>() {
            @Override
            public void subscribe(Subscriber<? super R> sub) {
                pub.subscribe(new DelegateSub<T,R>(sub) {
                    @Override
                    public void onNext(T i) {
                        sub.onNext(f.apply(i));
                    }
                });
            }
        };
    }

    private static <T> Subscriber<T> logSub() {
        return new Subscriber<>() {
            @Override
            public void onSubscribe(Subscription s) {
                System.out.println("onSubscribe");
                s.request(Long.MAX_VALUE);
            }

            @Override
            public void onNext(T i) {
                System.out.println("onNext:" + i);
            }

            @Override
            public void onError(Throwable t) {
                System.out.println("onError:" + t);
            }

            @Override
            public void onComplete() {
                System.out.println("onComplete:");
            }
        };
    }

    private static Publisher iterPub(Iterable<Integer> items) {
        return new Publisher() {
            @Override
            public void subscribe(Subscriber sub) {
                sub.onSubscribe(new Subscription() {
                    @Override
                    public void request(long n) {
                        try {
                            items.forEach(s -> sub.onNext(s));
                            sub.onComplete();
                        } catch (Throwable t) {
                            sub.onError(t);
                        }
                    }

                    @Override
                    public void cancel() {

                    }
                });
            }
        };
    }
}

public class DelegateSub<T,R> implements Subscriber<T> {
    Subscriber sub;

    public DelegateSub(Subscriber<? super R> sub) {
        this.sub = sub;
    }

    @Override
    public void onSubscribe(Subscription s) {
        sub.onSubscribe(s);
    }

    @Override
    public void onNext(T i) {
        sub.onNext(i);
    }

    @Override
    public void onError(Throwable t) {
        sub.onError(t);
    }

    @Override
    public void onComplete() {
        sub.onComplete();
    }
}

```

reactor.core에서는 위와 같은 publisher를 손쉽게 만들 수 있음.

```java
Flux.<Integer>create(e -> {
    e.next(1);
    e.next(2);
    e.next(3);
    e.complete();
})
.log()
.map(s -> s*10)
.log()
.subscribe(System.out::println);
}
```

# Schedulers

- 위의 예시는 pub/sub이 같은 쓰레드에 있다는 점이 문제가 된다.
- 즉, publish 시간과 subscribe 시간을 온전히 대기해야 하는 현상이 발생.
- `flux.subscribeOn()`, `flux.publishOn()` 같은 API는 별도의 쓰레드를 만들어서 처리할 수 있게 도와준다.
  - subscribeOn : 구독을 시작하는 시점부터 별도의 쓰레드를 생성해서 처리한다. (publish의 속도가 느린 경우에 적합하다.)
    - onSubscribe, request, onNext, onComplete가 전부 별도의 쓰레드에서 진행.
  - publishOn : 퍼블리싱을 하는 시점부터 별도의 쓰레드를 만들어서 처리한다. (subscribe의 속도가 느린 경우에 적합하다.)
    - onNext onComplete 시점부터 별도의 쓰레드에서 진행.

### 직접 구현한 Scheduler

```java
@Slf4j
public class ScheduleEx {
    public static void main(String[] args) {
        Publisher<Integer> pub = sub -> {
            sub.onSubscribe(new Subscription() {
                @Override
                public void request(long n) {
                    log.debug("request()");
                    sub.onNext(1);
                    sub.onNext(2);
                    sub.onNext(3);
                    sub.onNext(4);
                    sub.onNext(5);
                    sub.onComplete();
                }

                @Override
                public void cancel() {

                }
            });
        };

        Publisher<Integer> subOnPub = sub -> {
            ExecutorService es = Executors.newSingleThreadExecutor(new CustomizableThreadFactory("subOn-"));
            es.execute(() -> pub.subscribe(sub));
        };

        Publisher<Integer> pubOnPub = sub -> {
            pub.subscribe(new Subscriber<Integer>() {
                ExecutorService es = Executors.newSingleThreadExecutor(new CustomizableThreadFactory("pubOn-"));
                @Override
                public void onSubscribe(Subscription subscription) {
                    sub.onSubscribe(subscription);
                }

                @Override
                public void onNext(Integer item) {
                    es.execute(() -> sub.onNext(item));
                }

                @Override
                public void onError(Throwable throwable) {
                    es.execute(() -> sub.onError(throwable));
                    es.shutdown();      //다 끝났으므로 쓰레드 종료 shutdown은 graceful하게 종료.
                                        // shutdownNow는 강제로 인터럽트 걸어서 종료.
                }

                @Override
                public void onComplete() {
                    es.execute(() -> sub.onComplete());
                    es.shutdown(); //다 끝났으므로 쓰레드 종료
                }
            });
        };

        pubOnPub.subscribe(new Subscriber<Integer>() {
            @Override
            public void onSubscribe(Subscription subscription) {
                log.debug("onSubscribe");
                subscription.request(Long.MAX_VALUE);
            }

            @Override
            public void onNext(Integer item) {
                log.debug("onNext: {}", item);
            }

            @Override
            public void onError(Throwable throwable) {
                log.debug("onError: {}", throwable);
            }

            @Override
            public void onComplete() {
                log.debug("onComplete");
            }
        });

        System.out.println("exit");
    }
}

```

### PublishOn, SubscribeOn

```java
Flux.range(1, 10)
    .publishOn(Schedulers.newSingle("pub"))         //퍼블리싱을 하는 시점(onNext)부터 별도의 쓰레드 (consumer 느린경우)
    .log()
    .subscribeOn(Schedulers.newSingle("sub"))       //구독을 하는 시점(onSubscribe)부터 별도의 쓰레드 (producer가 느린경우)
    .subscribe(System.out::println);

System.out.println("exit");

```

- pub-1과 sub-1이 publish, subscribe 하도록 동작.


### 직접 구현한 take API

```java
        Publisher<Integer> pub = sub -> {
            sub.onSubscribe(new Subscription() {
                int i = 0;
                boolean canceled = false;
                @Override
                public void request(long n) {
                    ScheduledExecutorService exec = Executors.newSingleThreadScheduledExecutor();
                    exec.scheduleAtFixedRate(() -> {
                        if (canceled) {
                            exec.shutdown();
                            return;
                        }
                        sub.onNext(i++);
                    }, 0, 300, TimeUnit.MILLISECONDS);
                }

                @Override
                public void cancel() {
                    canceled = true;

                }
            });
        };

        Publisher<Integer> takePub = sub -> {
            pub.subscribe(new Subscriber<Integer>() {       //Proxy
                int count = 1;
                Subscription subscription;

                @Override
                public void onSubscribe(Subscription subscription) {
                    sub.onSubscribe(subscription);
                }

                @Override
                public void onNext(Integer item) {
                    sub.onNext(item);
                    if (++count > 10) {
                        subscription.cancel();
                    }
                }

                @Override
                public void onError(Throwable throwable) {
                    sub.onError(throwable);
                }

                @Override
                public void onComplete() {
                    sub.onComplete();
                }
            });
        };

        takePub.subscribe(new Subscriber<Integer>() {
            @Override
            public void onSubscribe(Subscription subscription) {
                log.debug("onSubscribe");
                subscription.request(Long.MAX_VALUE);
            }

            @Override
            public void onNext(Integer item) {
                log.debug("onNext: {}", item);
            }

            @Override
            public void onError(Throwable throwable) {
                log.debug("onError: {}", throwable);
            }

            @Override
            public void onComplete() {
                log.debug("onComplete");
            }
        });
    }
```


### Take API

```java
Flux.interval(Duration.ofMillis(200))
    .take(10)       //원하는 데이터의 개수만큼만 받을 수 있음. (paging처리 처럼)
    .subscribe(s -> log.debug("onNext:{}", s));

log.debug("exit");
TimeUnit.SECONDS.sleep(5);
```

- 별도의 쓰레드에서 실행 -> demon 쓰레드
- JVM은 user 쓰레드가 없고 데몬 쓰레드만 남아있으면 강제로 종료된다. 따라서 테스트를 위해 main쓰레드를 오래 살려놓기 위해 sleep을 주거나 해야 함.


