https://juejin.cn/post/7105968458851942414
## MemorySafeLinkedBlockingQueue
```java
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.apache.dubbo.common.threadpool;

import java.util.Collection;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;
/**
 * Can completely solve the OOM problem caused by {@link java.util.concurrent.LinkedBlockingQueue},
 * does not depend on {@link java.lang.instrument.Instrumentation} and is easier to use than
 * {@link MemoryLimitedLinkedBlockingQueue}.
 *
 * @see <a href="https://github.com/apache/incubator-shenyu/blob/master/shenyu-common/src/main/java/org/apache/shenyu/common/concurrent/MemorySafeLinkedBlockingQueue.java">MemorySafeLinkedBlockingQueue</a>
 */

public class MemorySafeLinkedBlockingQueue<E> extends LinkedBlockingQueue<E> {

  private static final long serialVersionUID = 8032578371739960142L;

  public static int THE_256_MB = 256 * 1024 * 1024;

  private int maxFreeMemory;

  public MemorySafeLinkedBlockingQueue() {
    this(THE_256_MB);
  }

  public MemorySafeLinkedBlockingQueue(final int maxFreeMemory) {
    super(Integer.MAX_VALUE);
    this.maxFreeMemory = maxFreeMemory;
  }

  public MemorySafeLinkedBlockingQueue(
    final Collection<? extends E> c,
    final int maxFreeMemory
  ) {
    super(c);
    this.maxFreeMemory = maxFreeMemory;
  }

  /**
   * set the max free memory.
   *
   * @param maxFreeMemory the max free memory
   */
  public void setMaxFreeMemory(final int maxFreeMemory) {
    this.maxFreeMemory = maxFreeMemory;
  }

  /**
   * get the max free memory.
   *
   * @return the max free memory limit
   */
  public int getMaxFreeMemory() {
    return maxFreeMemory;
  }

  /**
   * determine if there is any remaining free memory.
   *
   * @return true if has free memory
   */
  public boolean hasRemainedMemory() {
    return MemoryLimitCalculator.maxAvailable() > maxFreeMemory;
  }

  @Override
  public void put(final E e) throws InterruptedException {
    if (hasRemainedMemory()) {
      super.put(e);
    }
  }

  @Override
  public boolean offer(final E e, final long timeout, final TimeUnit unit)
    throws InterruptedException {
    return hasRemainedMemory() && super.offer(e, timeout, unit);
  }

  @Override
  public boolean offer(final E e) {
    return hasRemainedMemory() && super.offer(e);
  }
}

```

## MemoryLimitCalculator
```java
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.apache.dubbo.common.threadpool;

import java.lang.management.ManagementFactory;
import java.lang.management.MemoryMXBean;
import java.lang.management.MemoryUsage;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

/**
 * {@link javax.management.MXBean} technology is used to calculate the memory
 * limit by using the percentage of the current maximum available memory,
 * which can be used with {@link MemoryLimiter}.
 *
 * @see MemoryLimiter
 * @see <a href="https://github.com/apache/incubator-shenyu/blob/master/shenyu-common/src/main/java/org/apache/shenyu/common/concurrent/MemoryLimitCalculator.java">MemoryLimitCalculator</a>
 */
public class MemoryLimitCalculator {

    private static final MemoryMXBean MX_BEAN = ManagementFactory.getMemoryMXBean();

    private static volatile long maxAvailable;

    private static final ScheduledExecutorService SCHEDULER = Executors.newSingleThreadScheduledExecutor();
    static {
        // immediately refresh when this class is loaded to prevent maxAvailable from being 0
        refresh();
        // check every 50 ms to improve performance
        SCHEDULER.scheduleWithFixedDelay(MemoryLimitCalculator::refresh, 50, 50, TimeUnit.MILLISECONDS);
        Runtime.getRuntime().addShutdownHook(new Thread(SCHEDULER::shutdown));
    }

    private static void refresh() {
        final MemoryUsage usage = MX_BEAN.getHeapMemoryUsage();
        maxAvailable = usage.getCommitted();
    }

    /**
     * Get the maximum available memory of the current JVM.
     *
     * @return maximum available memory
     */
    public static long maxAvailable() {
        return maxAvailable;
    }

    /**
     * Take the current JVM's maximum available memory
     * as a percentage of the result as the limit.
     *
     * @param percentage percentage
     * @return available memory
     */
    public static long calculate(final float percentage) {
        if (percentage <= 0 || percentage > 1) {
            throw new IllegalArgumentException();
        }
        return (long) (maxAvailable() * percentage);
    }

    /**
     * By default, it takes 80% of the maximum available memory of the current JVM.
     *
     * @return available memory
     */
    public static long defaultLimit() {
        return (long) (maxAvailable() * 0.8);
    }
}
```


***


## MemoryLimitedLinkedBlockingQueue
```java
package org.apache.dubbo.common.threadpool.support.limited;

import java.lang.instrument.Instrumentation;
import java.util.Collection;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;

/**
 * Can completely solve the OOM problem caused by {@link java.util.concurrent.LinkedBlockingQueue}.
 */
public class MemoryLimitedLinkedBlockingQueue<E>
  extends LinkedBlockingQueue<E> {

  private static final long serialVersionUID = 1374792064759926198L;

  private final MemoryLimiter memoryLimiter;

  public MemoryLimitedLinkedBlockingQueue(Instrumentation inst) {
    this(Integer.MAX_VALUE, inst);
  }

  public MemoryLimitedLinkedBlockingQueue(
    long memoryLimit,
    Instrumentation inst
  ) {
    super(Integer.MAX_VALUE);
    this.memoryLimiter = new MemoryLimiter(memoryLimit, inst);
  }

  public MemoryLimitedLinkedBlockingQueue(
    Collection<? extends E> c,
    long memoryLimit,
    Instrumentation inst
  ) {
    super(c);
    this.memoryLimiter = new MemoryLimiter(memoryLimit, inst);
  }

  public void setMemoryLimit(long memoryLimit) {
    memoryLimiter.setMemoryLimit(memoryLimit);
  }

  public long getMemoryLimit() {
    return memoryLimiter.getMemoryLimit();
  }

  public long getCurrentMemory() {
    return memoryLimiter.getCurrentMemory();
  }

  public long getCurrentRemainMemory() {
    return memoryLimiter.getCurrentRemainMemory();
  }

  @Override
  public void put(E e) throws InterruptedException {
    memoryLimiter.acquireInterruptibly(e);
    super.put(e);
  }

  @Override
  public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
    return (
      memoryLimiter.acquire(e, timeout, unit) && super.offer(e, timeout, unit)
    );
  }

  @Override
  public boolean offer(E e) {
    return memoryLimiter.acquire(e) && super.offer(e);
  }

  @Override
  public E take() throws InterruptedException {
    final E e = super.take();
    memoryLimiter.releaseInterruptibly(e);
    return e;
  }

  @Override
  public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    final E e = super.poll(timeout, unit);
    memoryLimiter.releaseInterruptibly(e, timeout, unit);
    return e;
  }

  @Override
  public E poll() {
    final E e = super.poll();
    memoryLimiter.release(e);
    return e;
  }

  @Override
  public boolean remove(Object o) {
    final boolean success = super.remove(o);
    if (success) {
      memoryLimiter.release(o);
    }
    return success;
  }

  @Override
  public void clear() {
    super.clear();
    memoryLimiter.clear();
  }
}

```

## MemoryLimiter
```java
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.apache.dubbo.common.threadpool.support.limited;

import java.lang.instrument.Instrumentation;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.LongAdder;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/**
 * memory limiter.
 */
public class MemoryLimiter {

  private final Instrumentation inst;

  private long memoryLimit;

  private final LongAdder memory = new LongAdder();

  private final ReentrantLock acquireLock = new ReentrantLock();

  private final Condition notLimited = acquireLock.newCondition();

  private final ReentrantLock releaseLock = new ReentrantLock();

  private final Condition notEmpty = releaseLock.newCondition();

  public MemoryLimiter(Instrumentation inst) {
    this(Integer.MAX_VALUE, inst);
  }

  public MemoryLimiter(long memoryLimit, Instrumentation inst) {
    this.memoryLimit = memoryLimit;
    this.inst = inst;
  }

  public void setMemoryLimit(long memoryLimit) {
    if (memoryLimit <= 0) {
      throw new IllegalArgumentException();
    }
    this.memoryLimit = memoryLimit;
  }

  public long getMemoryLimit() {
    return memoryLimit;
  }

  public long getCurrentMemory() {
    return memory.sum();
  }

  public long getCurrentRemainMemory() {
    return getMemoryLimit() - getCurrentMemory();
  }

  private void signalNotEmpty() {
    releaseLock.lock();
    try {
      notEmpty.signal();
    } finally {
      releaseLock.unlock();
    }
  }

  private void signalNotLimited() {
    acquireLock.lock();
    try {
      notLimited.signal();
    } finally {
      acquireLock.unlock();
    }
  }

  /**
   * Locks to prevent both acquires and releases.
   */
  private void fullyLock() {
    acquireLock.lock();
    releaseLock.lock();
  }

  /**
   * Unlocks to allow both acquires and releases.
   */
  private void fullyUnlock() {
    releaseLock.unlock();
    acquireLock.unlock();
  }

  public boolean acquire(Object e) {
    if (e == null) {
      throw new NullPointerException();
    }
    if (memory.sum() >= memoryLimit) {
      return false;
    }
    acquireLock.lock();
    try {
      final long objectSize = inst.getObjectSize(e);
      if (memory.sum() + objectSize < memoryLimit) {
        memory.add(objectSize);
        if (memory.sum() < memoryLimit) {
          notLimited.signal();
        }
      }
    } finally {
      acquireLock.unlock();
    }
    if (memory.sum() > 0) {
      signalNotEmpty();
    }
    return true;
  }

  public void acquireInterruptibly(Object e) throws InterruptedException {
    if (e == null) {
      throw new NullPointerException();
    }
    final long objectSize = inst.getObjectSize(e);
    acquireLock.lockInterruptibly();
    try {
      while (memory.sum() + objectSize >= memoryLimit) {
        notLimited.await();
      }
      memory.add(objectSize);
      if (memory.sum() < memoryLimit) {
        notLimited.signal();
      }
    } finally {
      acquireLock.unlock();
    }
    if (memory.sum() > 0) {
      signalNotEmpty();
    }
  }

  public boolean acquire(Object e, long timeout, TimeUnit unit)
    throws InterruptedException {
    if (e == null) {
      throw new NullPointerException();
    }
    long nanos = unit.toNanos(timeout);
    final long objectSize = inst.getObjectSize(e);
    acquireLock.lockInterruptibly();
    try {
      while (memory.sum() + objectSize >= memoryLimit) {
        if (nanos <= 0) {
          return false;
        }
        nanos = notLimited.awaitNanos(nanos);
      }
      memory.add(objectSize);
      if (memory.sum() < memoryLimit) {
        notLimited.signal();
      }
    } finally {
      acquireLock.unlock();
    }
    if (memory.sum() > 0) {
      signalNotEmpty();
    }
    return true;
  }

  public void release(Object e) {
    if (null == e) {
      return;
    }
    if (memory.sum() == 0) {
      return;
    }
    final long objectSize = inst.getObjectSize(e);
    releaseLock.lock();
    try {
      if (memory.sum() > 0) {
        memory.add(-objectSize);
        if (memory.sum() > 0) {
          notEmpty.signal();
        }
      }
    } finally {
      releaseLock.unlock();
    }
    if (memory.sum() < memoryLimit) {
      signalNotLimited();
    }
  }

  public void releaseInterruptibly(Object e) throws InterruptedException {
    if (null == e) {
      return;
    }
    final long objectSize = inst.getObjectSize(e);
    releaseLock.lockInterruptibly();
    try {
      while (memory.sum() == 0) {
        notEmpty.await();
      }
      memory.add(-objectSize);
      if (memory.sum() > 0) {
        notEmpty.signal();
      }
    } finally {
      releaseLock.unlock();
    }
    if (memory.sum() < memoryLimit) {
      signalNotLimited();
    }
  }

  public void releaseInterruptibly(Object e, long timeout, TimeUnit unit)
    throws InterruptedException {
    if (null == e) {
      return;
    }
    final long objectSize = inst.getObjectSize(e);
    long nanos = unit.toNanos(timeout);
    releaseLock.lockInterruptibly();
    try {
      while (memory.sum() == 0) {
        if (nanos <= 0) {
          return;
        }
        nanos = notEmpty.awaitNanos(nanos);
      }
      memory.add(-objectSize);
      if (memory.sum() > 0) {
        notEmpty.signal();
      }
    } finally {
      releaseLock.unlock();
    }
    if (memory.sum() < memoryLimit) {
      signalNotLimited();
    }
  }

  public void clear() {
    fullyLock();
    try {
      if (memory.sumThenReset() < memoryLimit) {
        notLimited.signal();
      }
    } finally {
      fullyUnlock();
    }
  }
}

```