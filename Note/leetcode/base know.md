
#### 排序
	sort.Ints(nums)

#### 哈希表
1. 可以避免重复遍历
2. key一般是需要比较的值
3. value 一般是索引

#### 链表
1. 哨兵：dummy:=&ListNode{-1,nil}

#### 整数的一些基本用法
- 字符串转整数
	- `int(x - '0')`
- 整除
	- x/num
		- 用来判断两个数相加是否超过十，需要进1
- 取余数
	- x%num
- 逆转数字
	- `nums=123
	- `res=0`
	- `res=res*10+nums%10`
	- `nums=nums/10`

#### 滑动窗口
1. 边界判断
	1. 如果使用hashmap，left指针务必小于map里已存的right指针
		`index,ok:=map[s[r]];ok && index>=left`

#### 回文子串
1. 回文串判断
	1. 中心扩展法
	2. func expand(s string,l,r int) int
		1. 边界判断： for l>=0 && r<len(s) && s[l]= =s[r]
		2. 向两边扩展
			1. l--
			2. r++
		3. 计算此次回文串的长度
			1. r-l-1(因为前面l-1和r+1了)
	3. l,r为中心点
		1. 奇数中心点：L1=expand(s,i,i)
		2. 偶数中心点：L2=expand(s,i,i+1)
		3. 比较L1，L2的大小，取最大L并与缓存的res进行比较
		4. 更新start和end的结果
			1. start=i-(L-1)/2
			2. end=i+L/2
		5. 最终返回s[start:end+1]

#### 数字字母的变形
##### 场景
- Z字变形

##### 注意事项
2. 遍历原数据（字符串）
	1. 对数据的分列存储（[]string）
	2. 判断是否需要对上下边界进行方向转换
		1. 一般是对行列的上下边界判断
	3. 根据边界方向对行列进行加减
3. 最终聚合数据

#### 双指针
##### 盛最多的水
1. 对于在一个非排序的数组或者字符串去查找最值
2. 一般都是左右边界指针
3. 计算值然后比大小更新结果
4. 最后对左右指针进行大小比较，移动其中一端

#### 罗马数字的转换
1. 直接列举两个列表
	values := []int{1000, 900, 500, 400, 100, 90, 50, 40, 10, 9, 5, 4, 1}
	symbols := []string{"M", "CM", "D", "CD", "C", "XC", "L", "XL", "X", "IX", "V", "IV", "I"}
2. 然后遍历传入的数据
	1. 如果传入的是int
		1. for i:=0;i<len(values)&& nums>0;i++
		3. `for values[i]<nums{
		4. `res+=symbols[i]`
		5. `nums-=values[i]`
	2. 如果传入的是string
		1. `for i:=0;i<len(values)&& len(s)>0 ;i++{`
		2. `for strings.HasPrefix(s, symbols[i]){`
		3. `res+=values[i]`
		4. `s=s[len(symbols[i]):]`


#### 最长公共前缀
##### 思考
	对于求多个对象的共同点，可以先从两个对象的共同点进行着手

##### 解法
1. 求两个字符串的公共子串
	1. `n:=min(len(a),len(b))`
	2. `i:=0`
	3. `for i<n&&a[i]==b[i]`
	4. `i++`
	5. `return a[:i]`
2. 再求多个字符串的最长前缀
	1. `x:=len(s)`
	2. `prefix:=s[0]`
	3. `for i:=0;i<len(s);i++{`
	4. `prefix=commonPrefix(prefix,s[i])`
	5. `return prefix`

#### 三数之和
##### 思考
	对于求和问题，第一个想到的应该是双指针
	同时注意：
		- 双指针比较适合顺序排列，所以一般都会`sort.Ints(nums)`
		- 数组中可能会出现相邻重复的数字，需要加一个判断跳过
##### 解法
1. func(nums []int, target int)
2. 基础判断
	1. `if len(nums)<3{`
	2. `return nil`
3. 两层遍历
	1. 先是遍历最外层
	2. `for i:=0;i<len(nums)-2;i++`
	3. 定义双指针
	4. `l,r:=i+1,len(nums)`-1
	5. 然后遍历第二层
	6. `for l<r`
	7. 计算sum的值
	8. `sum:=nums[i]+nums[l]|nums[r]`
	9. 根据sum的值进行双指针收缩
	10. `if sum==target{return}`
	11. 因为数组排序过，可能存在重复的值，需要判断跳过
	12. `for l<r && nums[l]==nums[l+1]{l++}`
	13. `for l<r && nums[r]==nums[r-1]{r++}`
	14. 再根据sum和target的大小进行收缩l,r
	15. `else if sum<target{l++}`
	16. `else sum>target{r--}`

#### 字符串的匹配遍历可以用滑动窗口
s[i:i+n]== needStr

#### 去除重复数字
- i:=0
- for l:=0;l<len(s);l++
- if s[l]!=s[i]
- i++
- s[i]=s[l]