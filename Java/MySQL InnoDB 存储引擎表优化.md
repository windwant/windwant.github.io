
#一、InnoDB 表存储优化

###1、OPTIMIZE TABLE

适时的使用 OPTIMIZE TABLE 语句来重组表，压缩浪费的表空间。这是在其它优化技术不可用的情况下最直接的方法。OPTIMIZE TABLE 语句通过拷贝表数据并重建表索引，使得索引数据更加紧凑，减少空间碎片。语句的执行效果会因表的不同而不同。过大的表或者过大的索引及初次添加大量数据的情况下都会使得这一操作变慢。

###2、自增主键

InnoDB表，如果主键过长（长数据列做主键，或者多个列组合做主键）会浪费很多空间。同时，二级索引也包含主键。这种情况，可以考虑创建自增列作为主键，或者使用前缀索引。

###3、VARCHAR

对于需要存储长度不定或者包含很多NULL值的字符串列，使用 VARCHAR 代替 CHAR 。在小表应用上，缓存使用及磁盘 I/O 消耗会更小。

###4、压缩的行格式存储

对于包含大量重复文本或者数字的大表，可以考虑采用压缩的行格式存储。这样数据加载会减少对缓存及 I/O 的需求。在使用压缩行格式前，需要考虑压缩行格式 COMPRESSED 和的不同性能影响。

# 二、InnoDB 事务管理优化
优化 InnoDB 事务处理，主要需要找到事务特性和服务器负载间的某个平衡点。例如，一秒需要提交几千事务的，或者每隔2-3个小时提交一次事务的不同应用表现。

###1、AUTOCOMMIT 设置

MySQL 的默认设置 AUTOCOMMIT=1 会限制繁忙数据库的性能。如果可以的话，可以在应用中使用 SET AUTOCOMMIT=0 或者 START TRANSACTION ，然后将多个相关的数据变更操作添加到同一事务中，然后执行 COMMIT 语句来提交事务，提交数据变更。

InnoDB 对于引发数据库变更的操作，必须将其进行日志刷盘。

###2、只读事务

对于只包含 SELECT 语句的事务，启用 AUTOCOMMIT ，使得 InnoDB 能够识别只读事务，然后进行相应的优化。

###3、回滚操作

避免对大数据量操作插入，更新和删除之后的回滚操作。如果一个大的事务拖慢了服务器，那么回滚将是服务器性能变得更糟。可以分批处理大数据量操作。通过杀进程方式终止的回滚操作会在服务器启动时重新启动。
可以通过如下策略减少此类问题发生：

- 增大缓存，避免频繁磁盘I/O。

- 设置 innodb_change_buffering=all，这样 update 和 delete 操作也会和 insert 一样进行缓存，回滚也更快。

- 手动commit，分割大数据操作。

为了避免时空的回滚。增大缓存，使得回滚进程可以应用到最大的资源以便快速执行。或者杀掉回滚进程，然后使用innodb_force_recovery=3选项重启。

对于较多执行耗时inserts, updates, 及 deletes 操作的服务器，确保innodb_change_buffering=all开启。

###4、日志刷盘

InnoDB 如会每秒刷盘一次日志，如果可以承受最新事务崩溃的数据损失，可以设置innodb_flush_log_at_trx_commit = 0。虽然日志的刷盘操作也不是保证的，同时也可以设置innodb_support_xa = 0，减少磁盘和二进制日志的同步操作。

Note

innodb_support_xa 已被弃用，将来版本会被移除。MySQL 5.7.10版本，InnoDB  XA事务的两阶段提交是默认支持的，不能设置禁用innodb_support_xa。

###5、耗时事务数据

行修改或删除后，行数据及 undo logs 在物理上并没有立刻被变更。即使在事务立刻提交后。旧数据会保持直到之前启动的事务或者并发执行的事务完成后。这样，这些事务可以一直访问到相关的旧数据。所以耗时的事务会阻止 InnoDB 清除其它相关事务的数据。

###6、关联删除

如果一个耗时的事务修改或者删除了某些行。那么其它使用这些数据的事务，如果事务级别设置在READ COMMITTED 或者 REPEATABLE READ 级别，则需要额外的处理来重建旧数据。

###7、关联查询

当一个耗时的事务修改了某个表，其它使用此表的事务将不会使用覆盖索引。如果二级索引包含比较新的PAGE_MAX_TRX_ID，或者某些记录被标记为已删除，InnoDB 可能需要使用聚簇索引来查询相应的记录。

覆盖索引查询（使用二级索引即可获得所需的数据，而不需要访问表数据）

# 三、InnoDB只读事务优化

InnoDB 可以避免给只读事务赋 transaction ID (TRX_ID )。事务ID只对执行写操作，或者含锁读操作，如 SELECT ... FOR UPDATE有用。去除不必要的事务ID，有助于减少每次读写操作必须访问的内部数据结构大小。

InnoDB 在以下情景能够识别只读操作：

- 事务以语句 START TRANSACTION READ ONLY 开始，这种情况下，数据变更操作会引发错误，事务仍会以只读性质运行：

  ERROR 1792 (25006): Cannot execute statement in a READ ONLY transaction.

  对于事务中的临时表可以进行任何操作。

- autocommit = on，并且事务只包含一个语句，且语句为没有使用FOR UPDATE 或者 LOCK IN SHARED MODE 的SELECT 语句。

- 事务以READ ONLY 选项开始。

  这样，对于读繁忙的应用，如报表应用，可以将一系列的查询语句综合到一个只读的事务中，或者在执行查询前设置 autocommit = on，或者在应用中避免将变更操作和查询操作相互影响。.

# 四、重做日志（redo log）优化

可以考虑遵循以下优化指引：

###1、日志大小

确保重做日志足够大，即使和缓存池（buffer pool）一样大。当 InnoDB 写满 redo log 时，服务器会基于一个检查点（checkpoint）就会将日志中的变更内容写到磁盘。如果 redo log 文件过小，那么就会引发服务器频繁的写盘。虽然之前，设置过大的 redo log 会引起恢复时间的过长，但是现在，恢复机制已经在速度上有很大的优化，因此不用再考虑此因素。
文件的大小和数量可以使用： innodb_log_file_size 和 innodb_log_files_in_group 进行设置。

###2、日志缓存

可以考虑增大日志缓存（log buffer）。大的日志缓存可以容纳更大的事务执行，避免不必要的写盘操作。设置变量：innodb_log_buffer_size 。

###3、read-on-write

配置 innodb_log_write_ahead_size 变量以避免 “read-on-write”（就是当修改的字节不足一个系统 block时，需要将整个 block 读进内存，修改对应的位置，然后再写进去；如果我们以 block 为单位来写入的话，直接完整覆盖写入即可）。这个配置定义了 redo log 的 write-ahead 块大小。

设置innodb_log_write_ahead_size 的大小以匹配操作系统或者文件系统的缓存块大小。

Read-on-write 的产生是因为在 write-ahead 块大小和操作系统或者文件系统的缓存块大小不匹配的情况下，redo log 块无法完全的写入到操作系统，或者文件系统引起的。

innodb_log_write_ahead_size 的值可以设置为 InnoDB 日志文件块大小的倍数(2n)。最小的值为(512)。设置为最小值时 Write-ahead 不会发生。最大值为 innodb_page_size 。如果设置的值大于innodb_page_size，那么服务器会使用innodb_page_size值。

innodb_log_write_ahead_size 值设置的太小，会导致 read-on-write；设置过大，则会影响 fsync 性能，因为一次需要些多个数据块。

# 五、InnoDB表的大数据载入

快速插入通用指引：

###1、AUTOCOMMIT

导入数据时，关闭 autocommit 模式，避免每次行插入导致的日志刷盘。在执行开始及结束使用 SET AUTOCOMMIT 及 COMMIT 语句：

SET autocommit=0;
...SQLimport statements ...
COMMIT;

mysqldump 选项 --opt （默认启用）会创建 dump 转储 文件，以执行快速数据导入，避免将所有的数据载入内存引发问题。即使不使用SET autocommit 和 COMMIT。

###2、二级索引键 UNIQUE 限制

如果在二级索引键上有 UNIQUE 限制，可以在载入时暂时关闭此检查：

SET unique_checks=0;
...SQLimport statements ...
SET unique_checks=1;

对于较大的表，此操作可以节省大量的磁盘 I/O，因为 InnoDB 可以使用它的 change buffer（change buffer 的主要目的是将对二级索引的数据操作缓存下来，以此减少二级索引的随机IO，并达到操作合并的效果）来批量写二级索引记录。确保数据不包含重复键。

###3、FOREIGN KEY

如果表键包含 FOREIGN KEY 限制。可以再导入期间关闭此限制。

###4、批量多行插入

使用批量多行插入，以减少不必要客户端服务器间通信：

SET foreign_key_checks=0;
...SQLimport statements ...
SET foreign_key_checks=1;

INSERT INTO yourtable VALUES (1,2), (5,5), ...;

适用于任何类型表。

###5、自增列批量插入

当批量插入涉及自增列时，设置 innodb_autoinc_lock_mode = 2 （默认1，0：traditional；1：consecutive；2：interleaved）。

###6、主键顺序插入

以主键的顺序进行批量插入会更快。InnoDB 表主键索引为聚簇索引（clustered index, 以主键的顺序访问会很快）。特别是对于无法完全载入缓存的大表。

###7、全文索引

全文索引导入：

- 表创建时定义新列FTS_DOC_ID，类型 BIGINT UNSIGNED NOT NULL,，列上定义索引FTS_DOC_ID_INDEX，如下：

  CREATE TABLE t1 (

  FTS_DOC_ID BIGINT unsigned NOT NULL AUTO_INCREMENT,

  title varchar(255) NOT NULL DEFAULT '',

  text mediumtext NOT NULL,

  PRIMARY KEY ('FTS_DOC_ID')

  ) ENGINE=InnoDB DEFAULT CHARSET=latin1;CREATE UNIQUE INDEX FTS_DOC_ID_INDEX on t1(FTS_DOC_ID);

- 载入数据。

- 数据载入后，在相应的列上创建全文索引。

  Note

  在 innodb 存储引擎中，为了支持全文检索，必须有一个列与 word 进行映射，在 innodb 中这个列被命名为FTS_DOC_ID，其类型必须为 BIGINT UNSIGNED NOT NULL,并且 innodb 存储引擎会在该列上加上一个名为FTS_DOC_ID_INDEX 的唯一索引。上述操作由 innodb 存储引擎自己完成，用户也可以在创建表时手动添加，主要对应的约束条件。

常规的索引是文档到关键词的映射：文档——>关键词

倒排索引是关键词到文档的映射：关键词——>文档

全文索引通过关键字找到关键字所在文档，可以提高查询效率

# 六、InnoDB 查询优化

创建适当的索引以优化查询，通用指引如下：

- 将关键查询最常用的的列包含进表主键中。

- 主键列不要使用过多的列或者过长的列。因为二级索引包含主键，过大的主键会造成磁盘I/O及内存的浪费。

- 不要在每个列上创建二级索引，一个查询只能使用一个索引。对于极少使用的列及列选择性不大的列创建索引对于查询优化不会有太大帮助。如果针对一个表的查询非常多，则需要找到能够有助于最多查询的多列主键。如果索引列能够覆盖所需要查询的数据列，那么就可以只使用索引进行数据查询，而不需要从表中获取数据。

- 如果某一列的数据不能为NULL，那么在创建表的时候将其生命为 NOT NULL 。优化器以此可以更高的决定最优使用索引。

- 可以针对但查询事务进行相应的优化。

# 七、InnoDB DDL 操作优化

- 许多DDL操作，如表和索引的(CREATE, ALTER, 及DROP 语句) 可以在线执行。

- 使用 TRUNCATE TABLE 代替 DELETE FROM tbl_name 来清空表，外键限制可以使得 TRUNCATE 语句如普通的 DELETE 语句般操作。这种情景下，一系列如 DROP TABLE 及 CREATE TABLE 语句会执行的很快。

- 因为主键InnoDB表的存储结构是高度整合的，主键的变更会引起整张表的重构。最好将主键定义包含在表创建语句中，避免不必要的后期更改。

# 八、InnoDB 磁盘 I/O 优化

如果数据库的设计及 sql 操作优化都遵循了最佳实践，数据库依然因为 I/O 负载而反应非常慢，那么就需要针对 I/O 进行专门的优化。可以通过 Unix 的 top 工具，或者 Windows 的任务管理器来查看工作负载，如果低 于70%，那么负载则主要在磁盘。

### 1、增大缓存：

InnoDB 缓存中的数据访问不需要磁盘I/O，使用innodb_buffer_pool_size 设置。建议设置为系统内存的 50 ~ 75%。

### 2、调整刷盘策略：

某些版本 GNU/Linux 系统，使用fsync() 或者相关方法进行刷盘时，速度会非常慢。这时，如果影响到数据库性能，那么可以通过设置innodb_flush_method = O_DSYNC 来变更刷盘策略。

### 3、调整 Linux 系统 AIO 磁盘调度策略为 noop（单队列） 或者deadline（读、写队列）

InnoDB 在 Linux 系统上使用异步 I/O 子系统（本地AIO）通过预读和写请求来执行数据文件读写。配置变量innodb_use_native_aio 默认启用。磁盘调度策略对本地 AIO 影响比较大。通常建议设置为 noop 或者 deadline。

### 4、Solaris 10 x86_64架构建议使用direct I/O策略

### 5、RAID配置

### 6、non-rotational 存储

Non-rotational 存储适用于随机读写；rotational 存储相反适用于顺序读写。不同的存储设备对数据及日志的操作类型不同。

数据库随机读写类文件包括：file-per-table 和 general tablespace 数据文件, undo tablespace文件和temporary tablespace 文件。顺序读写类文件包括：InnoDB system tablespace 文件(基于 doublewrite buffering and change buffering) 及日志文件（ binary log 文件和redo log 文件等）。

使用 non-rotational 存储时，需要对以下配置进行优化：

- innodb_checksum_algorithm：crc32 算法使用了一种更快的一致性检查算法，对于高速存储设备，推荐使用。

- innodb_flush_neighbors：针对rotational存储设备优化。对于non-rotational设备或者混用情景，则需禁用。

- innodb_io_capacity：默认的200设定对于低端non-rotational存储设备已经足够。其它，酌情设置。

- innodb_io_capacity_max：默认2000 针对non-rotational 存储。

- innodb_log_compressed_pages：如果redo logs存储在non-rotational设备，可以开率禁用词选项来减少日志。

- innodb_log_file_size：如果redo logs 存储在non-rotational 存储设备，设置此选项最大读写缓存。

- innodb_page_size：设置此值以匹配磁盘internal sector size。早期的SSD设备为4KB，一些新版本的SSD能够支持到16KB。默认的额InnoDB 也大小为16KB。尽量使得数据库页大小和存储设备的块大小接近，减少无法一次写入磁盘的数据大小。

- binlog_row_image：binary logs 存储在non-rotational 设备情况下，如果所有的表都有主键，那么可以将此变量设置为最小来减少日志。

### 7、增大 I/O 容量以减少 backlogs 负载

如果吞吐量会因为检测点操作而不间断的降低，那么可以开率增加 innodb_io_capacity 的值。值越大，数据库刷盘频率会增大，从而避免了因为 backlog 的操作带来的吞吐量的影响。

### 8、调整数据库I/O容量

如果系统能够满足 InnoDB 刷盘操作。可以考虑减小innodb_io_capacity 配置。通常需要将此变量尽量设置低一些。（SHOW ENGINE INNODB STATUS）

### 9、将系统表空间文件存储在 Fusion-io设备

如果使用支持原子写的 Fusion-io 设备存储系统表空间文件(“ibdata files”) ，那么可以对 doublewrite buffer-related I/O进行相应的优化。这种情况下，会自动使用 Fusion-io 设备的原子写替代 doublewrite buffering (innodb_doublewrite)进行数据的读写。这种特性只支持 Fusion-io 硬件设备及 Fusion-io NVMFS Linux 应用。可以通过变量 innodb_flush_method = O_DIRECT 进行配置。

Note

设置是全局性的，影响所有设备上的数据读写。

### 10、禁用压缩数据页日志

使用 InnoDB 表压缩特性时，重新压缩的图片数据页，如果数据有变化，则会写入 redo log。配置变量innodb_log_compressed_pages 默认启用，防止数据库恢复期间，因为 zlib 算法的变化引发数据库崩溃。如果可以确认 zlib 版本不会发生变化，那么可以关闭 innodb_log_compressed_pages 变量来减少重压缩产生的 redo log 负载。

