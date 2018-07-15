---
title: Glide 源码分析(2) - 内存缓存和数组缓存
date: 2018-06-20 14:59:29
categories: Android
tags: Glide
description: 在上篇我们了解了 Glide 的硬盘缓存，现在，我们继续看他的内存缓存和数组缓存。
---

在[上篇](/2018/06/08/glide-disk-cache/)我们了解了 Glide 的硬盘缓存，现在，我们继续看他的内存缓存和数组缓存。

## MemoryCache

### 相关接口

首先，虽然没有什么意思，出于完整性，我们还是先看看 `MemoryCache` 这个接口：
```Java
/**
 * An interface for adding and removing resources from an in memory cache.
 */
public interface MemoryCache {
  /**
   * An interface that will be called whenever a bitmap is removed from the cache.
   */
  interface ResourceRemovedListener {
    void onResourceRemoved(@NonNull Resource<?> removed);
  }

  /**
   * Returns the sum of the sizes of all the contents of the cache in bytes.
   */
  long getCurrentSize();

  /**
   * Returns the current maximum size in bytes of the cache.
   */
  long getMaxSize();

  /**
   * Adjust the maximum size of the cache by multiplying the original size of the cache by the given
   * multiplier.
   *
   * <p> If the size multiplier causes the size of the cache to be decreased, items will be evicted
   * until the cache is smaller than the new size. </p>
   *
   * @param multiplier A size multiplier >= 0.
   */
  void setSizeMultiplier(float multiplier);

  /**
   * Removes the value for the given key and returns it if present or null otherwise.
   *
   * @param key The key.
   */
  @Nullable
  Resource<?> remove(@NonNull Key key);

  /**
   * Add bitmap to the cache with the given key.
   *
   * @param key      The key to retrieve the bitmap.
   * @param resource The {@link com.bumptech.glide.load.engine.EngineResource} to store.
   * @return The old value of key (null if key is not in map).
   */
  @Nullable
  Resource<?> put(@NonNull Key key, @Nullable Resource<?> resource);

  /**
   * Set the listener to be called when a bitmap is removed from the cache.
   *
   * @param listener The listener.
   */
  void setResourceRemovedListener(@NonNull ResourceRemovedListener listener);

  /**
   * Evict all items from the memory cache.
   */
  void clearMemory();

  /**
   * Trim the memory cache to the appropriate level. Typically called on the callback onTrimMemory.
   *
   * @param level This integer represents a trim level as specified in {@link
   *              android.content.ComponentCallbacks2}.
   */
  void trimMemory(int level);
}
```
Glide 将内存缓存里的缓存实例抽象为 `Resource`。

```Java
/**
 * A resource interface that wraps a particular type so that it can be pooled and reused.
 *
 * @param <Z> The type of resource wrapped by this class.
 */
public interface Resource<Z> {

  /**
   * Returns the {@link Class} of the wrapped resource.
   */
  @NonNull
  Class<Z> getResourceClass();

  /**
   * Returns an instance of the wrapped resource.
   *
   * <p> Note - This does not have to be the same instance of the wrapped resource class and in fact
   * it is often appropriate to return a new instance for each call. For example,
   * {@link android.graphics.drawable.Drawable Drawable}s should only be used by a single
   * {@link android.view.View View} at a time so each call to this method for Resources that wrap
   * {@link android.graphics.drawable.Drawable Drawable}s should always return a new
   * {@link android.graphics.drawable.Drawable Drawable}. </p>
   */
  @NonNull
  Z get();

  /**
   * Returns the size in bytes of the wrapped resource to use to determine how much of the memory
   * cache this resource uses.
   */
  int getSize();

  /**
   * Cleans up and recycles internal resources.
   *
   * <p> It is only safe to call this method if there are no current resource consumers and if this
   * method has not yet been called. Typically this occurs at one of two times:
   * <ul>
   *   <li>During a resource load when the resource is transformed or transcoded before any consumer
   *   have ever had access to this resource</li>
   *   <li>After all consumers have released this resource and it has been evicted from the cache
   *   </li>
   * </ul>
   *
   * For most users of this class, the only time this method should ever be called is during
   * transformations or transcoders, the framework will call this method when all consumers have
   * released this resource and it has been evicted from the cache. </p>
   */
  void recycle();
}
```

### 实现

和大多数接口一样，`MemoryCache` 也有一个“傻瓜实现” `MemoryCacheAdapter`。这个类虽然实现了接口，但其实什么也没干。当我们想要完全关闭内存缓存的时候，就可以使用它。具体的代码这里就不看了。

`MemoryCache` 的真正实现是 `LruResourceCache`，它继承了 `LruCache`。在看 `LruResourceCache` 之前，我们先看看 `LruCache`。
```Java
/**
 * A general purpose size limited cache that evicts items using an LRU algorithm. By default every
 * item is assumed to have a size of one. Subclasses can override {@link #getSize(Object)}} to
 * change the size on a per item basis.
 *
 * @param <T> The type of the keys.
 * @param <Y> The type of the values.
 */
public class LruCache<T, Y> {
  // LinkedHashMap 的最后一个参数是 accessOrder，设置为 true 后，每次我们访问一个
  // 元素，对应的元素都会被移到链表的末尾
  private final Map<T, Y> cache = new LinkedHashMap<>(100, 0.75f, true);
  private final long initialMaxSize;
  private long maxSize;
  private long currentSize;

  /**
   * Constructor for LruCache.
   *
   * @param size The maximum size of the cache, the units must match the units used in {@link
   *             #getSize(Object)}.
   */
  public LruCache(long size) {
    this.initialMaxSize = size;
    this.maxSize = size;
  }

  /**
   * Sets a size multiplier that will be applied to the size provided in the constructor to put the
   * new size of the cache. If the new size is less than the current size, entries will be evicted
   * until the current size is less than or equal to the new size.
   *
   * @param multiplier The multiplier to apply.
   */
  public synchronized void setSizeMultiplier(float multiplier) {
    if (multiplier < 0) {
      throw new IllegalArgumentException("Multiplier must be >= 0");
    }
    maxSize = Math.round(initialMaxSize * multiplier);
    evict();
  }

  /**
   * Returns the size of a given item, defaulting to one. The units must match those used in the
   * size passed in to the constructor. Subclasses can override this method to return sizes in
   * various units, usually bytes.
   *
   * @param item The item to get the size of.
   */
  protected int getSize(@Nullable Y item) {
    return 1;
  }

  /**
   * Returns the number of entries stored in cache.
   */
  protected synchronized int getCount() {
    return cache.size();
  }

  /**
   * A callback called whenever an item is evicted from the cache. Subclasses can override.
   *
   * @param key  The key of the evicted item.
   * @param item The evicted item.
   */
  protected void onItemEvicted(@NonNull T key, @Nullable Y item) {
    // optional override
  }

  /**
   * Returns the current maximum size of the cache in bytes.
   */
  public synchronized long getMaxSize() {
    return maxSize;
  }

  /**
   * Returns the sum of the sizes of all items in the cache.
   */
  public synchronized long getCurrentSize() {
    return currentSize;
  }

  /**
   * Returns true if there is a value for the given key in the cache.
   *
   * @param key The key to check.
   */

  public synchronized boolean contains(@NonNull T key) {
    return cache.containsKey(key);
  }

  /**
   * Returns the item in the cache for the given key or null if no such item exists.
   *
   * @param key The key to check.
   */
  @Nullable
  public synchronized Y get(@NonNull T key) {
    return cache.get(key);
  }

  /**
   * Adds the given item to the cache with the given key and returns any previous entry for the
   * given key that may have already been in the cache.
   *
   * <p>If the size of the item is larger than the total cache size, the item will not be added to
   * the cache and instead {@link #onItemEvicted(Object, Object)} will be called synchronously with
   * the given key and item.
   *
   * @param key  The key to add the item at.
   * @param item The item to add.
   */
  @Nullable
  public synchronized Y put(@NonNull T key, @Nullable Y item) {
    final int itemSize = getSize(item);
    if (itemSize >= maxSize) {
      onItemEvicted(key, item);
      return null;
    }

    if (item != null) {
      currentSize += itemSize;
    }
    @Nullable final Y old = cache.put(key, item);
    if (old != null) {
      currentSize -= getSize(old);

      if (!old.equals(item)) {
        onItemEvicted(key, old);
      }
    }
    evict();

    return old;
  }

  /**
   * Removes the item at the given key and returns the removed item if present, and null otherwise.
   *
   * @param key The key to remove the item at.
   */
  @Nullable
  public synchronized Y remove(@NonNull T key) {
    final Y value = cache.remove(key);
    if (value != null) {
      currentSize -= getSize(value);
    }
    return value;
  }

  /**
   * Clears all items in the cache.
   */
  public void clearMemory() {
    trimToSize(0);
  }

  /**
   * Removes the least recently used items from the cache until the current size is less than the
   * given size.
   *
   * @param size The size the cache should be less than.
   */
  protected synchronized void trimToSize(long size) {
    Map.Entry<T, Y> last;
    Iterator<Map.Entry<T, Y>> cacheIterator;
    while (currentSize > size) {
      cacheIterator  = cache.entrySet().iterator();
      last = cacheIterator.next();
      final Y toRemove = last.getValue();
      currentSize -= getSize(toRemove);
      final T key = last.getKey();
      cacheIterator.remove();
      onItemEvicted(key, toRemove);
    }
  }

  private void evict() {
    trimToSize(maxSize);
  }
}
```

整个 `LruCache` 的实现，唯一需要注意的就是开头那一句代码，我们把 `accessOrder` 设置为 `true`。其余的都很简单，读者自己看一看就好。

接下来，我们看看更简单的 `LruResourceCache`：
```Java
/**
 * An LRU in memory cache for {@link com.bumptech.glide.load.engine.Resource}s.
 */
public class LruResourceCache extends LruCache<Key, Resource<?>> implements MemoryCache {
  private ResourceRemovedListener listener;

  /**
   * Constructor for LruResourceCache.
   *
   * @param size The maximum size in bytes the in memory cache can use.
   */
  public LruResourceCache(long size) {
    super(size);
  }

  @Override
  public void setResourceRemovedListener(@NonNull ResourceRemovedListener listener) {
    this.listener = listener;
  }

  @Override
  protected void onItemEvicted(@NonNull Key key, @Nullable Resource<?> item) {
    if (listener != null && item != null) {
      listener.onResourceRemoved(item);
    }
  }

  @Override
  protected int getSize(@Nullable Resource<?> item) {
    if (item == null) {
      return super.getSize(null);
    } else {
      return item.getSize();
    }
  }

  @SuppressLint("InlinedApi")
  @Override
  public void trimMemory(int level) {
    if (level >= android.content.ComponentCallbacks2.TRIM_MEMORY_BACKGROUND) {
      // Entering list of cached background apps
      // Evict our entire bitmap cache
      clearMemory();
    } else if (level >= android.content.ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN
        || level == android.content.ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL) {
      // The app's UI is no longer visible, or app is in the foreground but system is running
      // critically low on memory
      // Evict oldest half of our bitmap cache
      trimToSize(getMaxSize() / 2);
    }
  }
}
```
鉴于这两个类的实现是在是没有太多可圈可点的地方，为了不让读者白看一场，这里还是简单讲一下 LRU 的实现好了。

为了实现 LRU，关键的有两点：
1. 为了快速查找，我们需要一个 hash map
2. 访问元素后，我们需要把元素标记为“最新”。并且需要一种机制，可以让我们按访问时间依次遍历元素。这个时候，我们就需要一个双端链表。

把 hash map 和 double linked list 组合起来，就可以实现一个 LRU 了。这里使用的是 JDK 实现的 `LinkedHashMap`。


## ArrayPool

现在我们看看 `ArrayPool`。`ArrayPool` 的主要作用是缓存 `int[]` 和 `byte[]`。

### 接口

```Java
/**
 * Interface for an array pool that pools arrays of different types.
 */
public interface ArrayPool {
  /**
   * A standard size to use to increase hit rates when the required size isn't defined.
   * Currently 64KB.
   */
  int STANDARD_BUFFER_SIZE_BYTES = 64 * 1024;

  /**
   * Optionally adds the given array of the given type to the pool.
   *
   * <p>Arrays may be ignored, for example if the array is larger than the maximum size of the
   * pool.
   *
   * @deprecated Use {@link #put(Object)}
   */
  @Deprecated
  <T> void put(T array, Class<T> arrayClass);

  /**
   * Optionally adds the given array of the given type to the pool.
   *
   * <p>Arrays may be ignored, for example if the array is larger than the maximum size of the
   * pool.
   */
  <T> void put(T array);

  /**
   * Returns a non-null array of the given type with a length >= to the given size.
   *
   * <p>If an array of the given size isn't in the pool, a new one will be allocated.
   *
   * <p>This class makes no guarantees about the contents of the returned array.
   *
   * @see #getExact(int, Class)
   */
  <T> T get(int size, Class<T> arrayClass);

  /**
   * Returns a non-null array of the given type with a length exactly equal to the given size.
   *
   * <p>If an array of the given size isn't in the pool, a new one will be allocated.
   *
   * <p>This class makes no guarantees about the contents of the returned array.
   *
   * @see #get(int, Class)
   */
  <T> T getExact(int size, Class<T> arrayClass);

  /**
   * Clears all arrays from the pool.
   */
  void clearMemory();

  /**
   * Trims the size to the appropriate level.
   *
   * @param level A trim specified in {@link android.content.ComponentCallbacks2}.
   */
  void trimMemory(int level);

}
```

### 实现

在看 `ArrapPool` 的实现前，我们先看他用到的辅助类 `ArrayAdapterInterface` 和 `GroupedLinkedMap`。

#### ArrayAdapterInterface

前面我们提过，`ArrayPool` 的作用是缓存 `int[]` 和 `byte[]`。 `ArrayAdapterInterface` 的作用，就是隔离这两者的差别。
```Java
/**
 * Interface for handling operations on a primitive array type.
 * @param <T> Array type (e.g. byte[], int[])
 */
interface ArrayAdapterInterface<T> {

  /**
   * TAG for logging.
   */
  String getTag();

  /**
   * Return the length of the given array.
   */
  int getArrayLength(T array);

  /**
   * Allocate and return an array of the specified size.
   */
  T newArray(int length);

  /**
   * Return the size of an element in the array in bytes (e.g. for int return 4).
   */
  int getElementSizeInBytes();
}


public final class IntegerArrayAdapter implements ArrayAdapterInterface<int[]> {
  private static final String TAG = "IntegerArrayPool";

  @Override
  public String getTag() {
    return TAG;
  }

  @Override
  public int getArrayLength(int[] array) {
    return array.length;
  }

  @Override
  public int[] newArray(int length) {
    return new int[length];
  }

  @Override
  public int getElementSizeInBytes() {
    return 4;
  }
}
```

跟 `byte[]` 相对应的是 `ByteArrayAdapter`，他的实现跟 `IntegerArrayAdapter` 基本是一样的，这里就不看了。


#### GroupedLinkedMap

```Java
/**
 * Similar to {@link java.util.LinkedHashMap} when access ordered except that it is access ordered
 * on groups of bitmaps rather than individual objects. The idea is to be able to find the LRU
 * bitmap size, rather than the LRU bitmap object. We can then remove bitmaps from the least
 * recently used size of bitmap when we need to reduce our cache size.
 *
 * For the purposes of the LRU, we count gets for a particular size of bitmap as an access, even if
 * no bitmaps of that size are present. We do not count addition or removal of bitmaps as an
 * access.
 */
class GroupedLinkedMap<K extends Poolable, V> {
  // head 是链表的头节点
  private final LinkedEntry<K, V> head = new LinkedEntry<>();
  private final Map<K, LinkedEntry<K, V>> keyToEntry = new HashMap<>();

  public void put(K key, V value) {
    LinkedEntry<K, V> entry = keyToEntry.get(key);

    if (entry == null) {
      entry = new LinkedEntry<>(key);
      // 放到链表末尾
      makeTail(entry);
      keyToEntry.put(key, entry);
    } else {
      // 不需要用到这个 key。key 是 poolable 的，offer 后会放回 pool
      key.offer();
    }

    entry.add(value);
  }

  @Nullable
  public V get(K key) {
    LinkedEntry<K, V> entry = keyToEntry.get(key);
    if (entry == null) {
      entry = new LinkedEntry<>(key);
      keyToEntry.put(key, entry);
    } else {
      key.offer();
    }

    // 刚刚访问过的元素，要移到链表的开头
    makeHead(entry);

    // 如果是刚刚 new 出来的 LinkedEntry，这里会返回 null
    return entry.removeLast();
  }

  @Nullable
  public V removeLast() {
    // LinkedEntry 组成的是一个双向的链表，head.prev 是链表的尾节点
    LinkedEntry<K, V> last = head.prev;

    while (!last.equals(head)) {
      V removed = last.removeLast();
      if (removed != null) {
        return removed;
      } else {
        // We will clean up empty lru entries since they are likely to have been one off or
        // unusual sizes and
        // are not likely to be requested again so the gc thrash should be minimal. Doing so will
        // speed up our
        // removeLast operation in the future and prevent our linked list from growing to
        // arbitrarily large
        // sizes.
        removeEntry(last);
        keyToEntry.remove(last.key);
        last.key.offer();
      }

      last = last.prev;
    }

    return null;
  }

  @Override
  public String toString() {
    StringBuilder sb = new StringBuilder("GroupedLinkedMap( ");
    LinkedEntry<K, V> current = head.next;
    boolean hadAtLeastOneItem = false;
    while (!current.equals(head)) {
      hadAtLeastOneItem = true;
      sb.append('{').append(current.key).append(':').append(current.size()).append("}, ");
      current = current.next;
    }
    if (hadAtLeastOneItem) {
      sb.delete(sb.length() - 2, sb.length());
    }
    return sb.append(" )").toString();
  }

  // Make the entry the most recently used item.
  private void makeHead(LinkedEntry<K, V> entry) {
    removeEntry(entry);
    entry.prev = head;
    entry.next = head.next;
    updateEntry(entry);
  }

  // Make the entry the least recently used item.
  private void makeTail(LinkedEntry<K, V> entry) {
    removeEntry(entry);
    entry.prev = head.prev;
    entry.next = head;
    updateEntry(entry);
  }

  private static <K, V> void updateEntry(LinkedEntry<K, V> entry) {
    entry.next.prev = entry;
    entry.prev.next = entry;
  }

  private static <K, V> void removeEntry(LinkedEntry<K, V> entry) {
    entry.prev.next = entry.next;
    entry.next.prev = entry.prev;
  }

  private static class LinkedEntry<K, V> {
    @Synthetic final K key;
    private List<V> values;
    LinkedEntry<K, V> next;
    LinkedEntry<K, V> prev;

    // Used only for the first item in the list which we will treat specially and which will not
    // contain a value.
    LinkedEntry() {
      this(null);
    }

    LinkedEntry(K key) {
      next = prev = this;
      this.key = key;
    }

    @Nullable
    public V removeLast() {
      final int valueSize = size();
      return valueSize > 0 ? values.remove(valueSize - 1) : null;
    }

    public int size() {
      return values != null ? values.size() : 0;
    }

    public void add(V value) {
      if (values == null) {
        values = new ArrayList<>();
      }
      values.add(value);
    }
  }
}
```


#### LruArrayPool

`LruArrayPool` 是 `ArrayPool` 的实现。我们先看看如何把数组放到这个 pool 里，这个功能由 `put` 方法实现：
```Java
private final Map<Class<?>, NavigableMap<Integer, Integer>> sortedSizes = new HashMap<>();
private final Map<Class<?>, ArrayAdapterInterface<?>> adapters = new HashMap<>();

@Override
public synchronized <T> void put(T array) {
  @SuppressWarnings("unchecked")
  Class<T> arrayClass = (Class<T>) array.getClass();

  // getAdapterFromType 会跟进 class 返回 IntegerArrayAdapter 或 ByteArrayAdapter
  ArrayAdapterInterface<T> arrayAdapter = getAdapterFromType(arrayClass);
  int size = arrayAdapter.getArrayLength(array);
  int arrayBytes = size * arrayAdapter.getElementSizeInBytes();
  if (!isSmallEnoughForReuse(arrayBytes)) {
    return;
  }
  Key key = keyPool.get(size, arrayClass);

  groupedMap.put(key, array);
  NavigableMap<Integer, Integer> sizes = getSizesForAdapter(arrayClass);
  // key.size 其实就是数组长度
  Integer current = sizes.get(key.size);
  // TreeMap 里面记录的是每个 size 对应的数组有多少个
  sizes.put(key.size, current == null ? 1 : current + 1);
  currentSize += arrayBytes;
  evict();
}


@SuppressWarnings("unchecked")
private <T> ArrayAdapterInterface<T> getAdapterFromType(Class<T> arrayPoolClass) {
  ArrayAdapterInterface<?> adapter = adapters.get(arrayPoolClass);
  if (adapter == null) {
    if (arrayPoolClass.equals(int[].class)) {
      adapter = new IntegerArrayAdapter();
    } else if (arrayPoolClass.equals(byte[].class)) {
      adapter = new ByteArrayAdapter();
    } else {
        throw new IllegalArgumentException("No array pool found for: "
            + arrayPoolClass.getSimpleName());
    }
    adapters.put(arrayPoolClass, adapter);
  }
  return (ArrayAdapterInterface<T>) adapter;
}


private NavigableMap<Integer, Integer> getSizesForAdapter(Class<?> arrayClass) {
  // 一个数组类型对应一个 TreeMap（int[]/byte[] 分别对应一个 TreeMap）
  NavigableMap<Integer, Integer> sizes = sortedSizes.get(arrayClass);
  if (sizes == null) {
    sizes = new TreeMap<>();
    sortedSizes.put(arrayClass, sizes);
  }
  return sizes;
}
```

添加一个元素到缓存里，有可能使得缓存的总量超过了最大限度，所以在最后调用 `evict()`：
```Java
private void evict() {
  evictToSize(maxSize);
}

private void evictToSize(int size) {
  while (currentSize > size) {
    Object evicted = groupedMap.removeLast();
    Preconditions.checkNotNull(evicted);
    ArrayAdapterInterface<Object> arrayAdapter = getAdapterFromObject(evicted);
    currentSize -= arrayAdapter.getArrayLength(evicted) * arrayAdapter.getElementSizeInBytes();
    decrementArrayOfSize(arrayAdapter.getArrayLength(evicted), evicted.getClass());
    if (Log.isLoggable(arrayAdapter.getTag(), Log.VERBOSE)) {
      Log.v(arrayAdapter.getTag(), "evicted: " + arrayAdapter.getArrayLength(evicted));
    }
  }
}

private void decrementArrayOfSize(int size, Class<?> arrayClass) {
  NavigableMap<Integer, Integer> sizes = getSizesForAdapter(arrayClass);
  Integer current = sizes.get(size);
  if (current == null) {
    throw new NullPointerException(
        "Tried to decrement empty size" + ", size: " + size + ", this: " + this);
  }
  if (current == 1) {
    sizes.remove(size);
  } else {
    sizes.put(size, current - 1);
  }
}
```

这里的实现跟其他缓存差不多，就不说太多了。


下面我们看元素的获取。从缓存取出数组有两个方法，`getExact` 和 `get`。区别是，前者只会返回刚好符合所请求长度的数组，后者返回的数组则可能超过所请求长度。

```Java
@Override
public synchronized <T> T getExact(int size, Class<T> arrayClass) {
  Key key = keyPool.get(size, arrayClass);
  return getForKey(key, arrayClass);
}

private <T> T getForKey(Key key, Class<T> arrayClass) {
  ArrayAdapterInterface<T> arrayAdapter = getAdapterFromType(arrayClass);
  T result = getArrayForKey(key);
  if (result != null) {
    currentSize -= arrayAdapter.getArrayLength(result) * arrayAdapter.getElementSizeInBytes();
    decrementArrayOfSize(arrayAdapter.getArrayLength(result), arrayClass);
  }

  if (result == null) {
    if (Log.isLoggable(arrayAdapter.getTag(), Log.VERBOSE)) {
      Log.v(arrayAdapter.getTag(), "Allocated " + key.size + " bytes");
    }
    result = arrayAdapter.newArray(key.size);
  }
  return result;
}

// Our cast is safe because the Key is based on the type.
@SuppressWarnings({"unchecked", "TypeParameterUnusedInFormals"})
@Nullable
private <T> T getArrayForKey(Key key) {
  // groupedMap 的实现我们在前面看过了
  return (T) groupedMap.get(key);
}
```
`getExact` 的实现比较简单，反正要多少（长度）给多少就是了。下面我们看 `get` 方法。
```Java
@Override
public synchronized <T> T get(int size, Class<T> arrayClass) {
  // ceilingKey 的返回值大于等于 size
  // 这里就是我们要用 NavigableMap 的原因了，我们需要他的 ceilingKey()
  Integer possibleSize = getSizesForAdapter(arrayClass).ceilingKey(size);
  final Key key;
  if (mayFillRequest(size, possibleSize)) {
    key = keyPool.get(possibleSize, arrayClass);
  } else {
    key = keyPool.get(size, arrayClass);
  }
  return getForKey(key, arrayClass);
}

private boolean mayFillRequest(int requestedSize, Integer actualSize) {
  return actualSize != null
      && (isNoMoreThanHalfFull() || actualSize <= (MAX_OVER_SIZE_MULTIPLE * requestedSize));
}
```

因为缓存里可能没有我们需要的 size，但是有更大的。返回长度更大的对客户端并不会有什么影响，在某些情况下，还能增加缓存的利用率。但是，如果我们返回了一个更大的数组，就意味着有一部分空间客户端并不会使用，所以要对这个浪费的空间做一些限制。这个限制由 `MAX_OVER_SIZE_MULTIPLE` 控制。按照 4.7.1 版本的实现，这个值为 8，也就是说，可能会返回 8 倍于所请求长度的数组。

好了。`ArrayPool` 我们也看完啦。

<br>
