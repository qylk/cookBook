# 概念：
MMKV 是腾讯基于mmap内存映射的key-value组件，底层序列化/反序列化使用 protobuf 实现，性能高，稳定性强。功能上类似于ios上的NSUserDefaults或者android上的sharedPreference。

另外MMKV还支持加密存储、多进程共享。

# 开源地址：
https://github.com/tencent/mmkv

基本介绍：

[MMKV--基于 mmap 的 iOS 高性能通用 key-value 组件](https://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=2649286834&idx=1&sn=b58ec3323f415c36a8611079cae43d12&chksm=8334cc30b443452625f103b06667ba999c4c8cb8dc9ff7a7991ff2bd00c9748796d99eddd54f&scene=21#wechat_redirect)

# 原理分析：

# 源码解析：
1、初始化：
```java
//MMKV初始化入口
MMKV.initialize(this)
```
```java
public static String initialize(Context context) {
        rootDir = context.getFilesDir().getAbsolutePath() + "/mmkv";//工作目录，用于mmap映射的文件的根目录
        initialize(rootDir);//jni调用
        return rootDir;
}
```
```java
Java_com_tencent_mmkv_MMKV_initialize(JNIEnv *env, jobject obj, jstring rootDir) {
    if (!rootDir) {
        return;
    }
    const char *kstr = env->GetStringUTFChars(rootDir, nullptr);
    if (kstr) {
        MMKV::initializeMMKV(kstr);//c++层初始化
        env->ReleaseStringUTFChars(rootDir, kstr);
    }
}
```
```c++
void MMKV::initializeMMKV(const std::string &rootDir) {
    static pthread_once_t once_control = PTHREAD_ONCE_INIT;
    pthread_once(&once_control, initialize);//使用pthread_once保证只初始化一次，初始化工作在指定的第二个参数initialize方法中

    g_rootDir = rootDir;
    char *path = strdup(g_rootDir.c_str());
    mkPath(path);//生成工作目录
    free(path);
}

//一次初始化调用
void initialize() {
  	//堆内存上分配map数据结构，key是mmapID，value是MMKV实例指针，用于保存所有的MMKV数据。
  	//mmapID相当于一个sharedPreference的文件名，MMKV相当于一个sharedPreference实例。
    g_instanceDic = new unordered_map<std::string, MMKV *>;
    g_instanceLock = ThreadLock();//线程锁
}
```

2、实例化MMKV:
```java
/**
mmapID: 字符串型名字，随便起，只要不包含特殊字符
mode: 模式，可组合模式：
   SINGLE_PROCESS_MODE：单进程
   MULTI_PROCESS_MODE：多进程共享模式
   ASHMEM_MODE：匿名共享内存模式，默认是mmap方式的内存共享
cryptKey: 加密key   
*/
MMKV kv = MMKV.mmkvWithID(mmapID, mode, cryptKey);
```
```java
public static MMKV mmkvWithID(String mmapID, int mode, String cryptKey) {
        if (rootDir == null) {
            throw new IllegalStateException("You should Call MMKV.initialize() first.");
        } else {
            verifyMMID(mmapID);//检查id字符的合法性
          	//jni调用获取一个C层的MMKV对象句柄(可以理解为对象指针)，这个句柄用于java层的MMKV对象访问C层的mmkv对象。
            long handle = getMMKVWithID(mmapID, mode, cryptKey);
            return new MMKV(handle);
        }
}
```
```c++
const int DEFAULT_MMAP_SIZE = getpagesize();//默认的mmap长度 = 一个页面大小（32位系统中通常是4k字节）

extern "C" JNIEXPORT JNICALL jlong Java_com_tencent_mmkv_MMKV_getMMKVWithID(
    JNIEnv *env, jobject obj, jstring mmapID, jint mode, jstring cryptKey) {
    
  	MMKV *kv = nullptr;
    if (!mmapID) {
        return (jlong) kv;
    }
    string str = jstring2string(env, mmapID);

    if (cryptKey != nullptr) {
        string crypt = jstring2string(env, cryptKey);
        if (crypt.length() > 0) {//使用加密模式
          	//调用mmkvWithID方法new一个MMKV对象
            kv = MMKV::mmkvWithID(str, DEFAULT_MMAP_SIZE, (MMKVMode) mode, &crypt);
        }
    }
    if (!kv) {
        kv = MMKV::mmkvWithID(str, DEFAULT_MMAP_SIZE, (MMKVMode) mode, nullptr);//无加密模式
    }

    return (jlong) kv;
}
```
```c++
MMKV *MMKV::mmkvWithID(const std::string &mmapID, int size, MMKVMode mode, string *cryptKey) {
    ...
    auto itr = g_instanceDic->find(mmapID);//根据mmapId从map中查找
    if (itr != g_instanceDic->end()) {//找到了
        MMKV *kv = itr->second;
        return kv;//返回mmkv实例对象指针
    }
    auto kv = new MMKV(mmapID, size, mode, cryptKey);//没找到，new一个MMKV
    (*g_instanceDic)[mmapID] = kv;//存到map中，下次就能找到了
    return kv;//返回mmkv实例对象指针
}
```
```c++
MMKV::MMKV(const std::string &mmapID, int size, MMKVMode mode, string *cryptKey)
    : m_mmapID(mmapID)//mmapId
    , m_path(mappedKVPathWithID(m_mmapID, mode))//mmap映射文件路径
    , m_crcPath(crcPathWithID(m_mmapID, mode))//crc文件路径
    , m_metaFile(m_crcPath, DEFAULT_MMAP_SIZE, (mode & MMKV_ASHMEM) ? MMAP_ASHMEM : MMAP_FILE)//meta数据结构存储一些基本信息
    , m_crypter(nullptr)//加密器
    , m_fileLock(m_metaFile.getFd())//文件锁
    , m_sharedProcessLock(&m_fileLock, SharedLockType)
    , m_exclusiveProcessLock(&m_fileLock, ExclusiveLockType)
    , m_isInterProcess((mode & MMKV_MULTI_PROCESS) != 0)//多进程模式
    , m_isAshmem((mode & MMKV_ASHMEM) != 0) {//是否是匿名共享内存模式
    m_fd = -1;
    m_ptr = nullptr;
    m_size = 0;
    m_actualSize = 0;
    m_output = nullptr;

    if (m_isAshmem) {
        m_ashmemFile = new MmapedFile(m_mmapID, static_cast<size_t>(size), MMAP_ASHMEM);//打开匿名共享内存
        m_fd = m_ashmemFile->getFd();
    } else {
        m_ashmemFile = nullptr;
    }

    if (cryptKey && cryptKey->length() > 0) {
        m_crypter = new AESCrypt((const unsigned char *) cryptKey->data(), cryptKey->length());//初始化加密器
    }

    m_needLoadFromFile = true;

    m_crcDigest = 0;

    m_sharedProcessLock.m_enable = m_isInterProcess;
    m_exclusiveProcessLock.m_enable = m_isInterProcess;

    // sensitive zone
    {
        SCOPEDLOCK(m_sharedProcessLock);
        loadFromFile();
    }
}
```
```c++
//m_metaFile是MmapedFile类型的，path是crc文件，构造函数：
MmapedFile::MmapedFile(const std::string &path, size_t size, bool fileType)
    : m_name(path), m_fd(-1), m_segmentPtr(nullptr), m_segmentSize(0), m_fileType(fileType) {
    if (m_fileType == MMAP_FILE) {//mmap模式
        m_fd = open(m_name.c_str(), O_RDWR | O_CREAT, S_IRWXU);//读写模式打开文件
        if (m_fd < 0) {
            MMKVError("fail to open:%s, %s", m_name.c_str(), strerror(errno));
        } else {
            struct stat st = {};
            if (fstat(m_fd, &st) != -1) {
                m_segmentSize = static_cast<size_t>(st.st_size);
            }
            if (m_segmentSize < DEFAULT_MMAP_SIZE) {
                m_segmentSize = static_cast<size_t>(DEFAULT_MMAP_SIZE);
              	//ftruncate将crc文件大小调整为DEFAULT_MMAP_SIZE大小
                if (ftruncate(m_fd, m_segmentSize) != 0 || !zeroFillFile(m_fd, 0, m_segmentSize)) {
                    MMKVError("fail to truncate [%s] to size %zu, %s", m_name.c_str(),
                              m_segmentSize, strerror(errno));
                    close(m_fd);
                    m_fd = -1;
                    removeFile(m_name);
                    return;
                }
            }
          	//调用mmap映射crc文件到内存
            m_segmentPtr =
                (char *) mmap(nullptr, m_segmentSize, PROT_READ | PROT_WRITE, MAP_SHARED, m_fd, 0);
            if (m_segmentPtr == MAP_FAILED) {
                MMKVError("fail to mmap [%s], %s", m_name.c_str(), strerror(errno));
                close(m_fd);
                m_fd = -1;
                m_segmentPtr = nullptr;
            }
        }
    } else {//ashmem模式
        m_fd = open(ASHMEM_NAME_DEF, O_RDWR);
        if (m_fd < 0) {
            MMKVError("fail to open ashmem:%s, %s", m_name.c_str(), strerror(errno));
        } else {
            if (ioctl(m_fd, ASHMEM_SET_NAME, m_name.c_str()) != 0) {
                MMKVError("fail to set ashmem name:%s, %s", m_name.c_str(), strerror(errno));
            } else if (ioctl(m_fd, ASHMEM_SET_SIZE, size) != 0) {
                MMKVError("fail to set ashmem:%s, size %d, %s", m_name.c_str(), size,
                          strerror(errno));
            } else {
                m_segmentSize = static_cast<size_t>(size);
                m_segmentPtr = (char *) mmap(nullptr, m_segmentSize, PROT_READ | PROT_WRITE,
                                             MAP_SHARED, m_fd, 0);
                if (m_segmentPtr == MAP_FAILED) {
                    MMKVError("fail to mmap [%s], %s", m_name.c_str(), strerror(errno));
                    m_segmentPtr = nullptr;
                } else {
                    return;
                }
            }
            close(m_fd);
            m_fd = -1;
        }
    }
}
```

```c++
void MMKV::loadFromFile() {
    if (m_isAshmem) {
        loadFromAshmem();
        return;
    }

    m_metaInfo.read(m_metaFile.getMemory());

    m_fd = open(m_path.c_str(), O_RDWR | O_CREAT, S_IRWXU);//打开mmkv数据文件
    if (m_fd < 0) {
        MMKVError("fail to open:%s, %s", m_path.c_str(), strerror(errno));
    } else {
        m_size = 0;
        struct stat st = {0};
        if (fstat(m_fd, &st) != -1) {
            m_size = static_cast<size_t>(st.st_size);
        }
        // round up to (n * pagesize)
        if (m_size < DEFAULT_MMAP_SIZE || (m_size % DEFAULT_MMAP_SIZE != 0)) {
            size_t oldSize = m_size;
            m_size = ((m_size / DEFAULT_MMAP_SIZE) + 1) * DEFAULT_MMAP_SIZE;
            if (ftruncate(m_fd, m_size) != 0) {//将文件大小调整为n*pagesize
                MMKVError("fail to truncate [%s] to size %zu, %s", m_mmapID.c_str(), m_size,
                          strerror(errno));
                m_size = static_cast<size_t>(st.st_size);
            }
            zeroFillFile(m_fd, oldSize, m_size - oldSize);//用0填充剩余空间
        }
        //调用mmap映射数据文件到内存
        m_ptr = (char *) mmap(nullptr, m_size, PROT_READ | PROT_WRITE, MAP_SHARED, m_fd, 0);
        if (m_ptr == MAP_FAILED) {
            MMKVError("fail to mmap [%s], %s", m_mmapID.c_str(), strerror(errno));
        } else {
            memcpy(&m_actualSize, m_ptr, Fixed32Size);//提取前4个字节：数据长度
            MMKVInfo("loading [%s] with %zu size in total, file size is %zu", m_mmapID.c_str(),
                     m_actualSize, m_size);
            bool loaded = false;
            if (m_actualSize > 0) {
                if (m_actualSize < m_size && m_actualSize + Fixed32Size <= m_size) {
                    if (checkFileCRCValid()) {//校验crc文件
                        MMKVInfo("loading [%s] with crc %u sequence %u", m_mmapID.c_str(),
                                 m_metaInfo.m_crcDigest, m_metaInfo.m_sequence);
                        MMBuffer inputBuffer(m_ptr + Fixed32Size, m_actualSize, MMBufferNoCopy);//将所有数据加载到buffer中，非拷贝模式
                        if (m_crypter) {
                            decryptBuffer(*m_crypter, inputBuffer);//数据解密
                        }
                        m_dic = MiniPBCoder::decodeMap(inputBuffer);//protobuf解码成数据map
                        m_output = new CodedOutputData(m_ptr + Fixed32Size + m_actualSize,
                                                       m_size - Fixed32Size - m_actualSize);//构造数据写入类，以后使用m_output来写入数据
                        loaded = true;
                    }
                }
            }
            if (!loaded) {
                SCOPEDLOCK(m_exclusiveProcessLock);

                if (m_actualSize > 0) {
                    writeAcutalSize(0);
                }
                m_output = new CodedOutputData(m_ptr + Fixed32Size, m_size - Fixed32Size);
                recaculateCRCDigest();
            }
            MMKVInfo("loaded [%s] with %zu values", m_mmapID.c_str(), m_dic.size());
        }
    }

    if (!isFileValid()) {
        MMKVWarning("[%s] file not valid", m_mmapID.c_str());
    }

    m_needLoadFromFile = false;
}
```
3、数据写入：

//todo

4、数据读取：

//todo

