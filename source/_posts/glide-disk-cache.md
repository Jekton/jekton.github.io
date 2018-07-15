---
title: Glide 源码分析(1) - DiskCache 详解
date: 2018-06-08 18:32:03
categories: Android
tags: Glide
---


作为一个合格的图片加载框架，一般都会有内存缓存和硬盘缓存。在本篇，我们就先来看看 Glide 的硬盘缓存实现。

<!--more-->

> Glide 使用版本 4.7.1

## 接口说明

Glide 的硬盘缓存由接口 `DiskCache` 定义，用户可以根据需要，提供自己实现的 `DiskCache`，或者使用 Glide 内置的实现。

```Java
/**
 * An interface for writing to and reading from a disk cache.
 */
public interface DiskCache {

  /**
   * An interface for lazily creating a disk cache.
   */
  interface Factory {
    /** 250 MB of cache. */
    int DEFAULT_DISK_CACHE_SIZE = 250 * 1024 * 1024;
    String DEFAULT_DISK_CACHE_DIR = "image_manager_disk_cache";

    /** Returns a new disk cache, or {@code null} if no disk cache could be created. */
    @Nullable
    DiskCache build();
  }

  /**
   * An interface to actually write data to a key in the disk cache.
   */
  interface Writer {
    /**
     * Writes data to the file and returns true if the write was successful and should be committed,
     * and false if the write should be aborted.
     *
     * @param file The File the Writer should write to.
     */
    boolean write(@NonNull File file);
  }

  /**
   * Get the cache for the value at the given key.
   *
   * <p> Note - This is potentially dangerous, someone may write a new value to the file at any
   * point in time and we won't know about it. </p>
   *
   * @param key The key in the cache.
   * @return An InputStream representing the data at key at the time get is called.
   */
  @Nullable
  File get(Key key);

  /**
   * Write to a key in the cache. {@link Writer} is used so that the cache implementation can
   * perform actions after the write finishes, like commit (via atomic file rename).
   *
   * @param key    The key to write to.
   * @param writer An interface that will write data given an OutputStream for the key.
   */
  void put(Key key, Writer writer);

  /**
   * Remove the key and value from the cache.
   *
   * @param key The key to remove.
   */
  // Public API.
  @SuppressWarnings("unused")
  void delete(Key key);

  /**
   * Clear the cache.
   */
  void clear();
}
```

这个接口并没有太多值得玩味的地方，读者自己看一看就好。


## Factory 工厂

`DiskCache.Factory` 作为一个工厂，用来 build 出实际的 `DiskCache` 实例。Glide 内部共有 3 个 `Factory` 的实现，分别是：

```Java
class DiskLruCacheFactory implements DiskCache.Factory;
class InternalCacheDiskCacheFactory extends DiskLruCacheFactory;
class ExternalPreferredCacheDiskCacheFactory extends DiskLruCacheFactory;
```

`InternalCacheDiskCacheFactory` 所创建的 `DiskCache` 将文件存储在 internal storage，存放在这里的文件只有我们自己能够读取。
```Java
public final class InternalCacheDiskCacheFactory extends DiskLruCacheFactory {

  public InternalCacheDiskCacheFactory(Context context) {
    // 默认情况下，文件夹名字是 image_manager_disk_cache
    // 缓存大小为 250M
    this(context, DiskCache.Factory.DEFAULT_DISK_CACHE_DIR,
        DiskCache.Factory.DEFAULT_DISK_CACHE_SIZE);
  }

  public InternalCacheDiskCacheFactory(Context context, long diskCacheSize) {
    this(context, DiskCache.Factory.DEFAULT_DISK_CACHE_DIR, diskCacheSize);
  }

  public InternalCacheDiskCacheFactory(final Context context, final String diskCacheName,
                                       long diskCacheSize) {
    super(new CacheDirectoryGetter() {
      @Override
      public File getCacheDirectory() {
        // 从这里可以看到，我们确实是用的 internal storage
        File cacheDirectory = context.getCacheDir();
        if (cacheDirectory == null) {
          return null;
        }
        if (diskCacheName != null) {
          return new File(cacheDirectory, diskCacheName);
        }
        return cacheDirectory;
      }
    }, diskCacheSize);
  }
}

```

`ExternalPreferredCacheDiskCacheFactory` 优先使用 external storage，在 external storage 不可写的情况下，会转而使用 internal storage。
```Java
/**
 * Creates an {@link com.bumptech.glide.disklrucache.DiskLruCache} based disk cache in the external
 * disk cache directory, which falls back to the internal disk cache if no external storage is
 * available. If ever fell back to the internal disk cache, will use that one from that moment on.
 *
 * <p><b>Images can be read by everyone when using external disk cache.</b>
 */
// Public API.
@SuppressWarnings({"unused", "WeakerAccess"})
public final class ExternalPreferredCacheDiskCacheFactory extends DiskLruCacheFactory {

  public ExternalPreferredCacheDiskCacheFactory(Context context) {
    this(context, DiskCache.Factory.DEFAULT_DISK_CACHE_DIR,
        DiskCache.Factory.DEFAULT_DISK_CACHE_SIZE);
  }

  public ExternalPreferredCacheDiskCacheFactory(Context context, long diskCacheSize) {
    this(context, DiskCache.Factory.DEFAULT_DISK_CACHE_DIR, diskCacheSize);
  }

  public ExternalPreferredCacheDiskCacheFactory(final Context context, final String diskCacheName,
                                                final long diskCacheSize) {
    super(new CacheDirectoryGetter() {
      @Nullable
      private File getInternalCacheDirectory() {
        File cacheDirectory = context.getCacheDir();
        if (cacheDirectory == null) {
          return null;
        }
        if (diskCacheName != null) {
          return new File(cacheDirectory, diskCacheName);
        }
        return cacheDirectory;
      }

      @Override
      public File getCacheDirectory() {
        File internalCacheDirectory = getInternalCacheDirectory();

        // Already used internal cache, so keep using that one,
        // thus avoiding using both external and internal with transient errors.
        if ((null != internalCacheDirectory) && internalCacheDirectory.exists()) {
          return internalCacheDirectory;
        }

        File cacheDirectory = context.getExternalCacheDir();

        // Shared storage is not available.
        if ((cacheDirectory == null) || (!cacheDirectory.canWrite())) {
          return internalCacheDirectory;
        }
        if (diskCacheName != null) {
          return new File(cacheDirectory, diskCacheName);
        }
        return cacheDirectory;
      }
    }, diskCacheSize);
  }
}
```

当使用 external storage 的时候，路径为 `/sdcard/Android/data/your.package.name/`，这里路径所有程序都可以访问。我们自己往里面写东西，也不需要声明权限，在应用被删除后，这个文件夹会被系统一起删除。

缓存不管是放在 internal storage 还是 external storage，唯一的差异，其实就是存放的路径不同。所以，`InternalCacheDiskCacheFactory`、`ExternalPreferredCacheDiskCacheFactory` 都只是用于指定一个路径，实际的 factory 的职责由他们共同的父类 `DiskLruCacheFactory` 来完成。

```Java
/**
 * Creates an {@link com.bumptech.glide.disklrucache.DiskLruCache} based disk cache in the specified
 * disk cache directory.
 *
 * <p>If you need to make I/O access before returning the cache directory use the {@link
 * DiskLruCacheFactory#DiskLruCacheFactory(CacheDirectoryGetter, long)} constructor variant.
 */
// Public API.
@SuppressWarnings("unused")
public class DiskLruCacheFactory implements DiskCache.Factory {
  private final long diskCacheSize;
  private final CacheDirectoryGetter cacheDirectoryGetter;

  /**
   * Interface called out of UI thread to get the cache folder.
   */
  public interface CacheDirectoryGetter {
    File getCacheDirectory();
  }

  // 我们可以不使用 InternalCacheDiskCacheFactory、ExternalPreferredCacheDiskCacheFactory
  // 直接通过这个接口构建 DiskLruCacheFactory。通过这种方式，我们可以指定任意的缓存路
  public DiskLruCacheFactory(final String diskCacheFolder, long diskCacheSize) {
    this(new CacheDirectoryGetter() {
      @Override
      public File getCacheDirectory() {
        return new File(diskCacheFolder);
      }
    }, diskCacheSize);
  }

  public DiskLruCacheFactory(final String diskCacheFolder, final String diskCacheName,
                             long diskCacheSize) {
    this(new CacheDirectoryGetter() {
      @Override
      public File getCacheDirectory() {
        return new File(diskCacheFolder, diskCacheName);
      }
    }, diskCacheSize);
  }

  /**
   * When using this constructor {@link CacheDirectoryGetter#getCacheDirectory()} will be called out
   * of UI thread, allowing to do I/O access without performance impacts.
   *
   * @param cacheDirectoryGetter Interface called out of UI thread to get the cache folder.
   * @param diskCacheSize        Desired max bytes size for the LRU disk cache.
   */
  // Public API.
  @SuppressWarnings("WeakerAccess")
  public DiskLruCacheFactory(CacheDirectoryGetter cacheDirectoryGetter, long diskCacheSize) {
    this.diskCacheSize = diskCacheSize;
    this.cacheDirectoryGetter = cacheDirectoryGetter;
  }

  @Override
  public DiskCache build() {
    File cacheDir = cacheDirectoryGetter.getCacheDirectory();

    if (cacheDir == null) {
      return null;
    }

    if (!cacheDir.mkdirs() && (!cacheDir.exists() || !cacheDir.isDirectory())) {
      return null;
    }

    // 调用 DiskLruCacheWrapper 的工厂方法构造一个 DiskLruCacheWrapper，DiskLruCacheWrapper
    // 是 glide 内建的 DiskCache 实现
    return DiskLruCacheWrapper.create(cacheDir, diskCacheSize);
  }
}

```

`Factory` 的实现很简单，到这里我们就看完了。下面我们看看 DiskLruCacheWrapper。


## wrapper 实现

Glide 的硬盘缓存，实际上上并不是自己实现的，而是使用别人实现好的 disklrucache。在 build.gradle 可以看到：
```Gradle
// library/build.gradle

dependencies {
    api project(':third_party:disklrucache')

    // ...
}
```

由于 `DiskCache` 是 Glide 自动定义的接口，disklrucache 并没有实现它，所以这里我们借助 `DiskLruCacheWrapper` 来解决接口的问题（设计模式迷此时估计会大喊：adapter 模式!）。


### DiskLruCacheWrapper 的构造

```Java
/**
 * Create a new DiskCache in the given directory with a specified max size.
 *
 * @param directory The directory for the disk cache
 * @param maxSize   The max size for the disk cache
 * @return The new disk cache with the given arguments
 */
@SuppressWarnings("deprecation")
public static DiskCache create(File directory, long maxSize) {
  return new DiskLruCacheWrapper(directory, maxSize);
}

/**
 * @deprecated Do not extend this class.
 */
@Deprecated
// Deprecated public API.
@SuppressWarnings({"WeakerAccess", "DeprecatedIsStillUsed"})
protected DiskLruCacheWrapper(File directory, long maxSize) {
  this.directory = directory;
  this.maxSize = maxSize;
  this.safeKeyGenerator = new SafeKeyGenerator();
}
```
`SafeKeyGenerator` 用于将一个 `Key` 转换为 `String` 类型的 key，它跟主干逻辑没有太多关系，这里我们就不看了。


### 获取文件

获取文件由 `get(Key)` 完成：
```Java
private DiskLruCache diskLruCache;


private synchronized DiskLruCache getDiskCache() throws IOException {
  // 懒加载
  // 注意，这里跟 double check 不一样，diskLruCache 并不需要是 volatile。具体原因这里就不展开谈了。
  if (diskLruCache == null) {
    diskLruCache = DiskLruCache.open(directory, APP_VERSION, VALUE_COUNT, maxSize);
  }
  return diskLruCache;
}

@Override
public File get(Key key) {
  String safeKey = safeKeyGenerator.getSafeKey(key);
  if (Log.isLoggable(TAG, Log.VERBOSE)) {
    Log.v(TAG, "Get: Obtained: " + safeKey + " for for Key: " + key);
  }
  File result = null;
  try {
    // It is possible that the there will be a put in between these two gets. If so that shouldn't
    // be a problem because we will always put the same value at the same key so our input streams
    // will still represent the same data.
    final DiskLruCache.Value value = getDiskCache().get(safeKey);
    if (value != null) {
      result = value.getFile(0);
    }
  } catch (IOException e) {
    if (Log.isLoggable(TAG, Log.WARN)) {
      Log.w(TAG, "Unable to get from disk cache", e);
    }
  }
  return result;
}
```


### 存储文件
```Java
private final DiskCacheWriteLocker writeLocker = new DiskCacheWriteLocker();


@Override
public void put(Key key, Writer writer) {
  // We want to make sure that puts block so that data is available when put completes. We may
  // actually not write any data if we find that data is written by the time we acquire the lock.
  String safeKey = safeKeyGenerator.getSafeKey(key);
  // 获取针对 safeKey 的锁
  writeLocker.acquire(safeKey);
  try {
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      Log.v(TAG, "Put: Obtained: " + safeKey + " for for Key: " + key);
    }
    try {
      // We assume we only need to put once, so if data was written while we were trying to get
      // the lock, we can simply abort.
      DiskLruCache diskCache = getDiskCache();
      Value current = diskCache.get(safeKey);
      if (current != null) {
        return;
      }

      DiskLruCache.Editor editor = diskCache.edit(safeKey);
      if (editor == null) {
        throw new IllegalStateException("Had two simultaneous puts for: " + safeKey);
      }
      try {
        File file = editor.getFile(0);
        if (writer.write(file)) {
          editor.commit();
        }
      } finally {
        // DiskLruCache 实现了日志系统，写文件作为一个事务来处理
        editor.abortUnlessCommitted();
      }
    } catch (IOException e) {
      if (Log.isLoggable(TAG, Log.WARN)) {
        Log.w(TAG, "Unable to put to disk cache", e);
      }
    }
  } finally {
    writeLocker.release(safeKey);
  }
}

```

写文件的时候，不同的文件直接的写并不会相互影响，`writeLocker.acquire(safeKey)` 中会给每个 key 分配一个锁。关于 `DiskCacheWriteLocker` 的实现，由于篇幅关系，这里就不看了。


### 删除文件、情况缓存

```Java
@Override
public void delete(Key key) {
  String safeKey = safeKeyGenerator.getSafeKey(key);
  try {
    getDiskCache().remove(safeKey);
  } catch (IOException e) {
    if (Log.isLoggable(TAG, Log.WARN)) {
      Log.w(TAG, "Unable to delete from disk cache", e);
    }
  }
}

@Override
public synchronized void clear() {
  try {
    getDiskCache().delete();
  } catch (IOException e) {
    if (Log.isLoggable(TAG, Log.WARN)) {
      Log.w(TAG, "Unable to clear disk cache or disk cache cleared externally", e);
    }
  } finally {
    // Delete can close the cache but still throw. If we don't null out the disk cache here, every
    // subsequent request will try to act on a closed disk cache and fail. By nulling out the disk
    // cache we at least allow for attempts to open the cache in the future. See #2465.
    resetDiskCache();
  }
}

private synchronized void resetDiskCache() {
  diskLruCache = null;
}
```
这两个方法是在没有什么值得说的。下面我们接着看 `DiskLruCache` 的实现。



## DiskLruCache 实现

在看代码之前，我们先来聊聊为什么需要日志。最简单的情况，我们想要把某个图片缓存起来，由于写硬盘是一个比较耗时的操作，我们把它放到一个后台线程来执行。假设在写文件的过程中，很不幸的，其他某个线程抛出了未捕获的异常，导致程序终止了。这时候，我们刚才想要保存的图片，其实才写了一半。在这种情况下，我们就需要一种机制，可以表明这个文件是损坏了的，在程序下次启动的时候移它。


为了实现日志（journal）系统，一般的思路是，比方说上面的情况，我们在保存文件的前，先往一个日志文件写入一条日志，说我们马上就要写某某某文件了。写完后，我们再来执行文件保存动作。保存成功后，我们再一次修改日志文件，说我们已经保存成功了。

这时候，万一我们在保存文件的过程中跪了，程序下次启动的时候，只要检查日志文件，就知道哪些文件是损坏的（最倒霉的情况是，我们保存成功，在修改日志的时候跪了。此时，正常的这个文件也会被清除，但是，对于一个缓存，这又有什么所谓呢）。


我们先看看 `DiskLruCache` 里面关于日志文件的一段注释，它对我们理解代码有非常大的帮助：
```
/*
 * This cache uses a journal file named "journal". A typical journal file
 * looks like this:
 *     libcore.io.DiskLruCache
 *     1
 *     100
 *     2
 *
 *     CLEAN 3400330d1dfc7f3f7f4b8d4d803dfcf6 832 21054
 *     DIRTY 335c4c6028171cfddfbaae1a9c313c52
 *     CLEAN 335c4c6028171cfddfbaae1a9c313c52 3934 2342
 *     REMOVE 335c4c6028171cfddfbaae1a9c313c52
 *     DIRTY 1ab96a171faeeee38496d8b330771a7a
 *     CLEAN 1ab96a171faeeee38496d8b330771a7a 1600 234
 *     READ 335c4c6028171cfddfbaae1a9c313c52
 *     READ 3400330d1dfc7f3f7f4b8d4d803dfcf6
 *
 * The first five lines of the journal form its header. They are the
 * constant string "libcore.io.DiskLruCache", the disk cache's version,
 * the application's version, the value count, and a blank line.
 *
 * Each of the subsequent lines in the file is a record of the state of a
 * cache entry. Each line contains space-separated values: a state, a key,
 * and optional state-specific values.
 *   o DIRTY lines track that an entry is actively being created or updated.
 *     Every successful DIRTY action should be followed by a CLEAN or REMOVE
 *     action. DIRTY lines without a matching CLEAN or REMOVE indicate that
 *     temporary files may need to be deleted.
 *   o CLEAN lines track a cache entry that has been successfully published
 *     and may be read. A publish line is followed by the lengths of each of
 *     its values.
 *   o READ lines track accesses for LRU.
 *   o REMOVE lines track entries that have been deleted.
 *
 * The journal file is appended to as cache operations occur. The journal may
 * occasionally be compacted by dropping redundant lines. A temporary file named
 * "journal.tmp" will be used during compaction; that file should be deleted if
 * it exists when the cache is opened.
 */
```

### DiskLruCache 实例的创建

上面看 `DiskLruCacheWrapper` 的时候我们就知道，创建 `DiskLruCache` 时使用的是它的静态方法 `open`：
```Java
/**
 * Opens the cache in {@code directory}, creating a cache if none exists
 * there.
 *
 * @param directory a writable directory
 * @param valueCount the number of values per cache entry. Must be positive.
 * @param maxSize the maximum number of bytes this cache should use to store
 * @throws IOException if reading or writing the cache directory fails
 */
public static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize)
    throws IOException {
  if (maxSize <= 0) {
    throw new IllegalArgumentException("maxSize <= 0");
  }
  if (valueCount <= 0) {
    throw new IllegalArgumentException("valueCount <= 0");
  }

  // If a bkp file exists, use it instead.
  File backupFile = new File(directory, JOURNAL_FILE_BACKUP);
  if (backupFile.exists()) {
    File journalFile = new File(directory, JOURNAL_FILE);
    // If journal file also exists just delete backup file.
    if (journalFile.exists()) {
      backupFile.delete();
    } else {
      renameTo(backupFile, journalFile, false);
    }
  }

  // Prefer to pick up where we left off.
  DiskLruCache cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
  if (cache.journalFile.exists()) {
    try {
      cache.readJournal();
      cache.processJournal();
      return cache;
    } catch (IOException journalIsCorrupt) {
      System.out
          .println("DiskLruCache "
              + directory
              + " is corrupt: "
              + journalIsCorrupt.getMessage()
              + ", removing");
      cache.delete();
    }
  }

  // Create a new empty cache.
  directory.mkdirs();
  cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
  cache.rebuildJournal();
  return cache;
}
```

可以看到，跟我们上面讲的差不多，在创建的时候，会读取日志文件，然后做一些清理工作。下面我们先看 `readJournal` 方法：
```Java
private void readJournal() throws IOException {
  StrictLineReader reader = new StrictLineReader(new FileInputStream(journalFile), Util.US_ASCII);
  try {
    String magic = reader.readLine();
    String version = reader.readLine();
    String appVersionString = reader.readLine();
    String valueCountString = reader.readLine();
    String blank = reader.readLine();
    // 判断一些文件头。关于文件头的格式，在上面那段注释中有说明
    if (!MAGIC.equals(magic)
        || !VERSION_1.equals(version)
        || !Integer.toString(appVersion).equals(appVersionString)
        || !Integer.toString(valueCount).equals(valueCountString)
        || !"".equals(blank)) {
      throw new IOException("unexpected journal header: [" + magic + ", " + version + ", "
          + valueCountString + ", " + blank + "]");
    }

    int lineCount = 0;
    while (true) {
      try {
        // 读取并处理一条日志记录
        readJournalLine(reader.readLine());
        lineCount++;
      } catch (EOFException endOfJournal) {
        break;
      }
    }
    // readJournalLine 会把已经 REMOVE 的 entry 移除掉，多出来的行数是冗余的(redundant)；
    // 并且，一个缓存项也可能对应多行（DIRTY/CLEAN/READ/...)，但其实只需要一个 CLEAN 就够了。
    // 当冗余的行数过多的时候，我们就需要清理日志文件
    redundantOpCount = lineCount - lruEntries.size();

    // 写了一半，应用就跪了
    // If we ended on a truncated line, rebuild the journal before appending to it.
    if (reader.hasUnterminatedLine()) {
      rebuildJournal();
    } else {
      journalWriter = new BufferedWriter(new OutputStreamWriter(
          new FileOutputStream(journalFile, true), Util.US_ASCII));
    }
  } finally {
    Util.closeQuietly(reader);
  }
}

private void readJournalLine(String line) throws IOException {
  int firstSpace = line.indexOf(' ');
  if (firstSpace == -1) {
    throw new IOException("unexpected journal line: " + line);
  }

  int keyBegin = firstSpace + 1;
  int secondSpace = line.indexOf(' ', keyBegin);
  final String key;
  if (secondSpace == -1) {
    key = line.substring(keyBegin);
    if (firstSpace == REMOVE.length() && line.startsWith(REMOVE)) {
      // 每个操作就会记录一个日志信息，当我们删除一个缓存项的时候，添加一个 REMOVE。
      // 既然缓存都已经 remove 掉了，这个 entry 也就就没有存在的必要的了
      // 在 REMOVE 前，一定会有对应的 DIRTY/CLEAN 等，所以此时这个 key 对应的 entry
      // 一定是存在的
      lruEntries.remove(key);
      return;
    }
  } else {
    key = line.substring(keyBegin, secondSpace);
  }

  // 一个 Entry 代表缓存中的一项
  Entry entry = lruEntries.get(key);
  if (entry == null) {
    entry = new Entry(key);
    lruEntries.put(key, entry);
  }

  if (secondSpace != -1 && firstSpace == CLEAN.length() && line.startsWith(CLEAN)) {
    String[] parts = line.substring(secondSpace + 1).split(" ");
    entry.readable = true;
    entry.currentEditor = null;
    entry.setLengths(parts);
  } else if (secondSpace == -1 && firstSpace == DIRTY.length() && line.startsWith(DIRTY)) {
    entry.currentEditor = new Editor(entry);
  } else if (secondSpace == -1 && firstSpace == READ.length() && line.startsWith(READ)) {
    // This work was already done by calling lruEntries.get().
  } else {
    throw new IOException("unexpected journal line: " + line);
  }
}
```

下面我们再看看 `rebuildJournal`：

```Java
/**
 * Creates a new journal that omits redundant information. This replaces the
 * current journal if it exists.
 */
private synchronized void rebuildJournal() throws IOException {
  if (journalWriter != null) {
    journalWriter.close();
  }

  Writer writer = new BufferedWriter(
      // 注意，这里我们写的是 tmp 文件
      new OutputStreamWriter(new FileOutputStream(journalFileTmp), Util.US_ASCII));
  try {
    writer.write(MAGIC);
    writer.write("\n");
    writer.write(VERSION_1);
    writer.write("\n");
    writer.write(Integer.toString(appVersion));
    writer.write("\n");
    writer.write(Integer.toString(valueCount));
    writer.write("\n");
    writer.write("\n");

    for (Entry entry : lruEntries.values()) {
      if (entry.currentEditor != null) {
        writer.write(DIRTY + ' ' + entry.key + '\n');
      } else {
        writer.write(CLEAN + ' ' + entry.key + entry.getLengths() + '\n');
      }
    }
  } finally {
    writer.close();
  }

  if (journalFile.exists()) {
    renameTo(journalFile, journalFileBackup, true);
  }
  renameTo(journalFileTmp, journalFile, false);
  journalFileBackup.delete();

  journalWriter = new BufferedWriter(
      new OutputStreamWriter(new FileOutputStream(journalFile, true), Util.US_ASCII));
}

private static void renameTo(File from, File to, boolean deleteDestination) throws IOException {
  if (deleteDestination) {
    deleteIfExists(to);
  }
  if (!from.renameTo(to)) {
    throw new IOException();
  }
}
```

这个方法很简单，就是先写一个文件头，然后把已有的 entry 一条一条写进去。

比较有意思的是后面那几个 rename。前面我们说，日志文件是为了在程序崩溃（还有内核崩溃，但这个比较少见）时恢复信息用的。当日志中冗余的行数太多的时候，我们也需要 rebuild 一下。在这里，我们同样需要写文件，万一在 rebuild journal 的时候跪了呢？！

解决办法就是，原来的日志文件不动，我们先把新的日志写到一个 tmp 文件里，等写完再用一个相对清理的 rename 来替换文件。由于 rename 的时候会删除原文件，这就存在一种可能，我们删除了 journal，而 rename tmp to journal 却失败了。此时，日志就丢失了。

所以，在把 tmp rename 为 journal 前，我们又把 journal rename 为 backup。在这种情况下，如果第一个 rename 成功，第二个失败，我们就能够通过这个 backup 来恢复原来的日志（在 `DiskLruCache` 的构造函数中，我们就检查了 backup 是否存在）。接着，如果第二个 rename 成功了，那么 backup 就没用了，于是删掉它。

`Entry` 的主要代码如下：
```Java
private final class Entry {
  private final String key;

  /** Lengths of this entry's files. */
  private final long[] lengths;

  /** Memoized File objects for this entry to avoid char[] allocations. */
  File[] cleanFiles;
  File[] dirtyFiles;

  /** True if this entry has ever been published. */
  private boolean readable;

  /** The ongoing edit or null if this entry is not being edited. */
  private Editor currentEditor;

  /** The sequence number of the most recently committed edit to this entry. */
  private long sequenceNumber;

  private Entry(String key) {
    this.key = key;
    this.lengths = new long[valueCount];
    cleanFiles = new File[valueCount];
    dirtyFiles = new File[valueCount];

    // The names are repetitive so re-use the same builder to avoid allocations.
    StringBuilder fileBuilder = new StringBuilder(key).append('.');
    int truncateTo = fileBuilder.length();
    for (int i = 0; i < valueCount; i++) {
        fileBuilder.append(i);
        cleanFiles[i] = new File(directory, fileBuilder.toString());
        fileBuilder.append(".tmp");
        dirtyFiles[i] = new File(directory, fileBuilder.toString());
        fileBuilder.setLength(truncateTo);
    }
  }
}
```
可以看到，对于 entry 中的每个数据项，都有对应 dirty/clean 两个文件。这个也是为了数据的一致性而存在的，后面我们会看到他们的作用。


读取完日志文件后，`DiskLruCache` 又调用 `processJournal` 来处理它：
```Java
/**
 * Computes the initial size and collects garbage as a part of opening the
 * cache. Dirty entries are assumed to be inconsistent and will be deleted.
 */
private void processJournal() throws IOException {
  deleteIfExists(journalFileTmp);
  for (Iterator<Entry> i = lruEntries.values().iterator(); i.hasNext(); ) {
    Entry entry = i.next();
    if (entry.currentEditor == null) { // 这个 entry 处于 CLEAN 状态
      for (int t = 0; t < valueCount; t++) {
        // size 是当前硬盘缓存占用的字节数
        size += entry.lengths[t];
      }
    } else {
      // DIRTY，说明上次有人写了一半就跪了。这种情况下，需要删除对应的文件
      entry.currentEditor = null;
      // valueCount 是我们在构造 DiskLruCache 时传递进来的，表示一个 entry 有多少个 value
      // 我们用它来缓存文件，一个 entry 对应一个文件，valueCount == 1
      for (int t = 0; t < valueCount; t++) {
        deleteIfExists(entry.getCleanFile(t));
        deleteIfExists(entry.getDirtyFile(t));
      }
      i.remove();
    }
  }
}

```

到这里，`DiskLruCache` 就构建完成了。


### 写文件到缓存

我们先来回顾一下 `DiskLruCacheWrapper` 中是怎么写文件的：
```Java
public void put(Key key, Writer writer) {
  // ...

  DiskLruCache.Editor editor = diskCache.edit(safeKey);
  if (editor == null) {
    throw new IllegalStateException("Had two simultaneous puts for: " + safeKey);
  }
  try {
    File file = editor.getFile(0);
    if (writer.write(file)) {
      editor.commit();
    }
  } finally {
    editor.abortUnlessCommitted();
  }

  // ...
}
```

基本上分为 4 步：
1. 获取一个 editor
2. 写文件
3. commit
4. 如果失败了，做一些清理工作

下面我们一个个来看。

1. 获取一个 editor

```Java
/**
 * Returns an editor for the entry named {@code key}, or null if another
 * edit is in progress.
 */
public Editor edit(String key) throws IOException {
  return edit(key, ANY_SEQUENCE_NUMBER);
}

private synchronized Editor edit(String key, long expectedSequenceNumber) throws IOException {
  checkNotClosed();
  Entry entry = lruEntries.get(key);
  if (expectedSequenceNumber != ANY_SEQUENCE_NUMBER && (entry == null
      || entry.sequenceNumber != expectedSequenceNumber)) {
    return null; // Value is stale.
  }
  if (entry == null) {
    // 新建一个缓存
    entry = new Entry(key);
    lruEntries.put(key, entry);
  } else if (entry.currentEditor != null) {
    return null; // Another edit is in progress.
  }

  Editor editor = new Editor(entry);
  entry.currentEditor = editor;

  // Flush the journal before creating files to prevent file leaks.
  journalWriter.append(DIRTY);
  journalWriter.append(' ');
  journalWriter.append(key);
  journalWriter.append('\n');
  journalWriter.flush();
  return editor;
}
```
这里验证了我上面说的，在实际写文件前，先写了一个日志（啊，当然，你也可以说我是先看了代码，才写了上面那段话。如果如果能够做到这个，也是很不错的。学习一个特定的知识点，把它泛化后，你就得到了一个通用的问题解决方案）。


2. 写文件

```Java
File file = editor.getFile(0);
writer.write(file);
```
写文件分两步，或者一个 file，然后 write 进去实际的数据。前面我们提到过，`valueCount` 代表每个缓存项中的数据数，我们只是存一个文件，所以这个 `getFile(0)` 获取第一个数据项（也是唯一的一个）。

```Java
public File getFile(int index) throws IOException {
  synchronized (DiskLruCache.this) {
    if (entry.currentEditor != this) {
        throw new IllegalStateException();
    }
    if (!entry.readable) {
        written[index] = true;
    }
    File dirtyFile = entry.getDirtyFile(index);
    if (!directory.exists()) {
        directory.mkdirs();
    }
    return dirtyFile;
  }
}
```

这里我们知道，写入的实际上是 dirty file。


3. commit

```Java
/**
 * Commits this edit so it is visible to readers.  This releases the
 * edit lock so another edit may be started on the same key.
 */
public void commit() throws IOException {
  // The object using this Editor must catch and handle any errors
  // during the write. If there is an error and they call commit
  // anyway, we will assume whatever they managed to write was valid.
  // Normally they should call abort.
  completeEdit(this, true);
  committed = true;
}


private synchronized void completeEdit(Editor editor, boolean success) throws IOException {
  Entry entry = editor.entry;
  if (entry.currentEditor != editor) {
    throw new IllegalStateException();
  }

  // If this edit is creating the entry for the first time, every index must have a value.
  if (success && !entry.readable) {
    for (int i = 0; i < valueCount; i++) {
      if (!editor.written[i]) {
        editor.abort();
        throw new IllegalStateException("Newly created entry didn't create value for index " + i);
      }
      if (!entry.getDirtyFile(i).exists()) {
        editor.abort();
        return;
      }
    }
  }

  for (int i = 0; i < valueCount; i++) {
    File dirty = entry.getDirtyFile(i);
    if (success) {
      if (dirty.exists()) {
        File clean = entry.getCleanFile(i);
        dirty.renameTo(clean);
        long oldLength = entry.lengths[i];
        long newLength = clean.length();
        entry.lengths[i] = newLength;
        size = size - oldLength + newLength;
      }
    } else {
      deleteIfExists(dirty);
    }
  }

  redundantOpCount++;
  entry.currentEditor = null;
  if (entry.readable | success) {
    entry.readable = true;
    journalWriter.append(CLEAN);
    journalWriter.append(' ');
    journalWriter.append(entry.key);
    journalWriter.append(entry.getLengths());
    journalWriter.append('\n');

    if (success) {
      entry.sequenceNumber = nextSequenceNumber++;
    }
  } else {
    // 写失败，对应的缓存项也不再有效
    lruEntries.remove(entry.key);
    journalWriter.append(REMOVE);
    journalWriter.append(' ');
    journalWriter.append(entry.key);
    journalWriter.append('\n');
  }
  journalWriter.flush();

  // 使用的总空间太多，或者冗余的日志太多
  if (size > maxSize || journalRebuildRequired()) {
    executorService.submit(cleanupCallable);
  }
}


private final Callable<Void> cleanupCallable = new Callable<Void>() {
  public Void call() throws Exception {
    synchronized (DiskLruCache.this) {
      if (journalWriter == null) {
        return null; // Closed.
      }
      trimToSize();
      if (journalRebuildRequired()) {
        rebuildJournal();
        redundantOpCount = 0;
      }
    }
    return null;
  }
};

private void trimToSize() throws IOException {
  while (size > maxSize) {
    Map.Entry<String, Entry> toEvict = lruEntries.entrySet().iterator().next();
    remove(toEvict.getKey());
  }
}
```

这里唯一需要注意的是，我们必须在 rename dirty to clean 成功后，才能写一个 CLEAN 日志。否则，如果我们写入 CLEAN 成功但 rename 却失败了，就会导致客户读取到错误的缓存数据。


4. 如果失败了，做一些清理工作

```Java
editor.abortUnlessCommitted();
```

```Java
/**
 * Aborts this edit. This releases the edit lock so another edit may be
 * started on the same key.
 */
public void abort() throws IOException {
  // 传入 false 会导致对应的 entry 被删除
  completeEdit(this, false);
}

public void abortUnlessCommitted() {
  if (!committed) {
    try {
      abort();
    } catch (IOException ignored) {
    }
  }
}
```

下面，我们再看读文件部分。

### 从缓存中读取文件

还是一样，我们先看看 `DiskLruCacheWrapper` 中对 `DiskLruCache` 的调用方式：
```Java
public File get(Key key) {
  // ...

  final DiskLruCache.Value value = getDiskCache().get(safeKey);
  if (value != null) {
    result = value.getFile(0);
  }

  // ...
}
```

```Java
/**
 * Returns a snapshot of the entry named {@code key}, or null if it doesn't
 * exist is not currently readable. If a value is returned, it is moved to
 * the head of the LRU queue.
 */
public synchronized Value get(String key) throws IOException {
  checkNotClosed();
  Entry entry = lruEntries.get(key);
  if (entry == null) {
    return null;
  }

  if (!entry.readable) {
    return null;
  }

  for (File file : entry.cleanFiles) {
      // A file must have been deleted manually!
      if (!file.exists()) {
          return null;
      }
  }

  redundantOpCount++;
  journalWriter.append(READ);
  journalWriter.append(' ');
  journalWriter.append(key);
  journalWriter.append('\n');
  if (journalRebuildRequired()) {
    executorService.submit(cleanupCallable);
  }

  return new Value(key, entry.sequenceNumber, entry.cleanFiles, entry.lengths);
}
```

这个方法比较简单，读者自己看一看就好。



### 删除一个缓存项

```Java
/**
 * Drops the entry for {@code key} if it exists and can be removed. Entries
 * actively being edited cannot be removed.
 *
 * @return true if an entry was removed.
 */
public synchronized boolean remove(String key) throws IOException {
  checkNotClosed();
  Entry entry = lruEntries.get(key);
  if (entry == null || entry.currentEditor != null) {
    return false;
  }

  for (int i = 0; i < valueCount; i++) {
    File file = entry.getCleanFile(i);
    if (file.exists() && !file.delete()) {
      throw new IOException("failed to delete " + file);
    }
    size -= entry.lengths[i];
    entry.lengths[i] = 0;
  }

  redundantOpCount++;
  journalWriter.append(REMOVE);
  journalWriter.append(' ');
  journalWriter.append(key);
  journalWriter.append('\n');

  lruEntries.remove(key);

  if (journalRebuildRequired()) {
    executorService.submit(cleanupCallable);
  }

  return true;
}
```
和前面其他文件况类似，这里我们需要先删除文件，然后再写日志。这样一来，即便文件删除成功而写日志失败，只要在下次读取时检测文件是否存在就可以了。如果反过来写日志成功但删除文件失败，则会导致缓存文件残留下来。

### 清空缓存

```Java
/**
 * Closes the cache and deletes all of its stored values. This will delete
 * all files in the cache directory including files that weren't created by
 * the cache.
 */
public void delete() throws IOException {
  close();
  // 所有的缓存文件都放在 directory 里，整个目录删掉就“清空”了
  Util.deleteContents(directory);
}

/** Closes this cache. Stored values will remain on the filesystem. */
public synchronized void close() throws IOException {
  if (journalWriter == null) {
    return; // Already closed.
  }
  for (Entry entry : new ArrayList<Entry>(lruEntries.values())) {
    if (entry.currentEditor != null) {
      entry.currentEditor.abort();
    }
  }
  trimToSize();
  journalWriter.close();
  journalWriter = null;
}
```

不容易啊，到这里，`DiskCache` 就算是讲完了。



---
## 2018.06.20 补几个面试遇到的问题

### 日志文件的作用

1. 应用崩溃后回收损坏的文件
2. 日志文件记录了访问的顺序，通过日志文件，我们可以重新建立 LRU


### 考虑日志文件对回收损坏文件的作用，不用日志文件能够解决吗？

不能，因为我们没有办法知道一个文件是否是有效文件。
