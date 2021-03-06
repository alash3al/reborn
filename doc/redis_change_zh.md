### redis 修改部分（增加若干指令） ###
--------------------------------

#####slotsinfo [start] [count]#####

+ 命令说明：获取 redis 中 slot 的个数以及每个 slot 的大小

+ 命令参数：缺省查询 [0, MAX_SLOT_NUM)

  - start - 起始的 slot 序号
  
	缺省 = 0
    
  - count - 查询的区间的大小，即查询范围为 [start, start + count)

	缺省 = MAX_SLOT_NUM
	
+ 返回结果：返回结果是 slotinfo 的 array；slotinfo 本身也是一个 array。

		response := []slotinfo{slot1, slot2, slot3, ...}        
		slotinfo := []int{slotnum, slotsize}

		其中：
			INT slotnum  : slot 序号
			INT slotsize : slot 内数据个数

+ 例如：

		localhost:6379> slotsinfo 0 1024
			1) 1) (integer) 579  
			   2) (integer) 1
			2) 1) (integer) 1017
		       2) (integer) 1                 

#####slotsdel slot1 [slot2 …]#####

+ 命令说明：删除 redis 中若干 slot 下的全部 key-value

+ 命令参数：接受至少 1 个 slotnum 作为参数

+ 返回结果：格式参见 slotsinfo，不同的是：slotsize 表示删除后剩余大小，通常为 0。

+ 例如：

		localhost:6379> slotsdel 579 1017
			1) 1) (integer) 579
			   2) (integer) 0
			2) 1) (integer) 1017
			   2) (integer) 0

####数据迁移####
---------------

**以下4个命令是一族命令：**

+ slotsmgrtslot - *O(1)*

	随机在某个 slot 下迁移一个 key-value 到目标机器

+ slotsmgrtone - *O(1)*

	将指定的 key-value 迁移到目标机

+ slotsmgrttagslot - *O(n)*

	随机在某个 slot 下选择一个 key，并将与之有相同 tag 的 key-value 对全部迁移到目标机

+ slotsmgrttagone - *O(n)*

	将与指定 key 具有相同 tag 的所有 key-value 对迁移到目标机
	

#####slotsmgrtslot host port timeout slot#####

+ 命令说明：随机选择 slot 下的 1 个 key-value 到迁移到目标机（同步 IO 操作）

	- 如果当前 slot 已经空了或者选择的 key 刚好过期，返回 0
    
	- 如果当前 slot 下面还有 key 则选择一个进行迁移
    
	- 同时返回当前 slot 剩余 key 的个数
    
	- 迁移过程在目标机器调用 slotsrestore 命令，迁移会 **覆盖旧值**
    
	
+ 命令参数：

	- host
	- port
		目标机

		redis 内部缓存到 host:port 的连接 30s，超时或错误则关闭
	
	- timeout - 操作超时，单位 ms
	
		过程需要 3 个同步操作：
		
		1. 建立连接（可被缓存优化）
			
		2. 发送 key-value 数据
			
		3. 接受目标机返回
		
		指令保证每个操作不超过 timeout
		
	- slot - 指定迁移的 slot 序号

+ 返回结果： 操作返回 int

		response := []int{succ,size}

		其中：
			INT succ : 表示迁移是否成功。
				0 表示当前 slot 已经空了（迁移成功个数=0）
				1 表示迁移一个 key 成功，并从本地删除（迁移成功个数=1）
			INT size : 表示 slot 下剩余 key 的个数

+ 例如：

		localhost:6379> set a 100            # set <a, 100>
			OK
		localhost:6379> slotsinfo            # slot 大小为 1
			1) 1) (integer) 579
			   2) (integer) 1
		localhost:6379> slotsmgrtslot 127.0.0.1 6380 100 579
			1) (integer) 1                   # 成功迁移 value
			2) (integer) 0                   # slot 剩余的 value 数量
		localhost:6379> slotsinfo
			(empty list or set)
		localhost:6379> slotsmgrtslot 127.0.0.1 6380 100 579 1
			1) (integer) 0                   # 成功成功个数为 0；当前 slot 已经空了
			2) (integer) 0                   # slot 剩余的 value 数量


#####slotsmgrtone host port timeout key#####

+ 命令说明：迁移 key 到目标机，与 slotsmgrtslot 相同

+ 命令参数：参见 slotsmgrtslot

+ 返回结果： 操作返回 整数 (int)

		response := int(succ)

		其中：
			INT succ : 与 slotsmgrtslot 相似

+ 例如：

		localhost:6379> set a 100            # set <a, 100>
			OK
		localhost:6379> slotsinfo
			1) 1) (integer) 579
			   2) (integer) 1
		localhost:6379> slotsmgrtone 127.0.0.1 6380 100 a
			(integer) 1                      # 迁移成功
		localhost:6379> slotsmgrtone 127.0.0.1 6380 100 a
			(integer) 0                      # 放弃迁移，本地已经不存在了

#####slotsmgrttagone host port timeout key#####

+ 命令说明：迁移与 key 有相同的 tag 的所有 key 到目标机

	- 当 key 中不包含合法 tag 时，命令退化为 slotsmgrtone
	
	- 当 key 中包含合法 tag 时，命令扫描对应 slot 下的所有 key，找到所有含有相同 tag 的 key-value，一次原子的迁移到目标机
	
	- **操作需要遍历整个 slot，复杂度** ***O(n)*** **，如果对应 key 不包含 tag 则退化为** ***O(1)***
	
+ 命令参数：参见 slotsmgrtone

+ 返回结果： 操作返回 整数 (int)

		response := int(succ)

		其中：
			INT succ : 表示成功迁移的 key 的个数。

+ 例如：

		localhost:6379> set a{tag} 100        # set <a{tag}, 100>
			OK
		localhost:6379> set b{tag} 100        # set <b{tag}, 100>
			OK
		localhost:6379> slotsmgrttagone 127.0.0.1 6380 1000 {tag}
			(integer) 2
		localhost:6379> scan 0                # 迁移成功，本地不存在了
			1) "0"
			2) (empty list or set)
		localhost:6380> scan 0                # 数据一次成功迁移到目标机
			1) "0"
			2) 1) "a{tag}"
			   2) "b{tag}"

#####slotsmgrttagslot host port timeout slot#####

+ 命令说明：与 slotsmgrtslot 对应的迁移指令

	- 其他说明参考 slotsmgrtslot 以及 slotsmgrttagone 的解释即可
	
#####slotsrestore key1 ttl1 val1 [key2 ttl2 val2 …]#####

+ 命令说明：该命令是对 redis-2.8 的 restore 命令的扩展

	- 可以对 restore 多个 key-value
	
	- 过程是原子的。
	
+ **备注：与 restore 不同的是，slotsrestore 只支持 replace，即一定** ***覆盖旧值*** **。如果旧值已经存在，那么只可能是 redis-slots 或者 proxy 的实现 bug，程序会通过 redisLog 打印一条冲突记录。**

####调试相关####
---------------
	
#####slotshashkey key1 [key2 …]#####

+ 命令说明：计算并返回给定 key 的 slot 序号

+ 命令参数：输入为 1 个或多个 key

+ 返回结果： 操作返回 array

		response := []int{slot1, slot2...}

		其中：
			INT slot : 表示对应 key 的 slot 序号，即 hash32(key) % NUM_OF_SLOTS

+ 例如：

		localhost:6379> slotshashkey a b c   # 计算 <a,b,c> 的 slot 序号
			1) (integer) 579
			2) (integer) 1017
			3) (integer) 879

#####slotscheck#####

+ 命令说明：对 redis 内的 slots 进行一致性检查，即满足如下两条

	- 每个 slot 中保存的 key 都能在 db 中找到对应的 val
	
	- 每个 db 中的 key 都能在对应的 slot 中查找到
	
+ 命令参数：0 参数

+ 返回结果： 操作返回 字符串 OK（如果 check 失败，会返回 ERR 并包含对应出错的 key）

+ 例如：

		localhost:6379> set a 100            # set <a, 100>
			OK
		localhost:6379> slotscheck
			OK                               # 检查通过
		…
		localhost:6379> slotscheck
			OK                               # 检查通过，但是耗时 1.07s
			(1.07s)

+ **备注**：***该操作比较慢，仅仅作为 redis 开发的调试工具使用，不能在线上使用***
