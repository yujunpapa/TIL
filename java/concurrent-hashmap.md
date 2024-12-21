
# ConcurrentHashMap의 size 구현방법

java ConcurrentHashMap은 성능향상을 위해서 lock striping방식으로 버킷단위로 lock을 건다. 버킷단위로 lock을 걸기때문에 `size()` 같은 메서드는 어떻게 동시성 처리를 했을까?

아래는 `size()` 메서드의 실제 구현내용이다. 내부 로직에서는 `sumCount()` 메서드를 호출한다. 이걸 통해서 추측한 내용은 put,putIfAbsent,remove등의 hashMap사이즈에 변경을 주는 opertaion진행시에 별도의 카운터값을 증가하거나 감소하는 형태임을 알 수 있다.


```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}

final long sumCount() {
    CounterCell[] cs = counterCells;
    long sum = baseCount;
    if (cs != null) {
        for (CounterCell c : cs)
            if (c != null)
                sum += c.value;
    }
    return sum;
}
```



여기서 `sumCount()` 메서드를 보면 CounterCell이라는 객체를 참조한다.  CounterCell은 내부에서 volatile로 선언한 value를 가지고 있는 단순한 객체이다.
```java

@jdk.internal.vm.annotation.Contended static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}

```


이제 CounterCell의 값을 변경시키는 addCount메서드를 확인해보자.

이 메서드는 스레드가 **baseCount**를 업데이트하려다 경쟁 상황을 감지하면, **CounterCells** 배열 중 하나의 카운터를 업데이트 하는 방식으로 concurrentHashMap의 사이즈를 변경시킨다.
```java
private final void addCount(long x, int check) {
    CounterCell[] cs; long b, s;
    
    //1. baseCount를 직접 업데이트
    if ((cs = counterCells) != null ||
        !U.compareAndSetLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell c; long v; int m;
        boolean uncontended = true;
        //2. counterCells 배열이 비어있거나, cas실패 fullAddCount호출
        if (cs == null || (m = cs.length - 1) < 0 ||
            (c = cs[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSetLong(c, CELLVALUE, v = c.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        //3. 맵의 크기를 합산하여 계산
        if (check <= 1)
            return;
        s = sumCount();
    }
    //4. ??
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n) << RESIZE_STAMP_SHIFT;
            if (sc < 0) {
                if (sc == rs + MAX_RESIZERS || sc == rs + 1 ||
                    (nt = nextTable) == null || transferIndex <= 0)
                    break;
                if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSetInt(this, SIZECTL, sc, rs + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}

```