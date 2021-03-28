# 变量
- 尽量用有意义的变量名, 多个单词用下划线分开
```
my_name = "coco"
my_score = 100
school_name = "回民小学"
```

- 变量可以重新赋值
```
student_name = "coco"
do_homework(student_name)

student_name = "bobo"
do_homework(student_name)
```

# 字符串
- 初始化
```
my_name = "coco"
```

- 获取长度
```
my_name = "coco"
name_len = len(my_name)
print(name_len) 
>>> 4
```

- 判断是否为空字符串
```
my_name = "coco"
if my_name:
  print("the string is not empty")
else:
  print("the string is empty")
```

- 遍历字符串里的字符
```
my_name = "coco"
for c in my_name:
  print(c)
>>> c
>>> o
>>> c
>>> o
```

- 格式化
```
message = "my name is {}, my score is {}, my favorite color is {}".format("coco", 100, "pink")
print(message)
>>> my name is coco, my score is 100, my favorite color is pink
```

# 输入输出
- 获取用户输入
```
# player_name为用户输入的字符串
player_name = input("what's your name?\n")
>>> player_name = "ABC"

# player_score为用户输入的字符串
player_score = input("what's your score?\n")
>>> player_score = "100"

# 先得到用户输入的字符串, 然后转化成int
player_score = int(input("what's your score?\n"))
>>> player_score = 100
```

- 打印字符串
```
# 直接打印字符串
print("Hello World")
>>> Hello World

# 打印字符串变量
msg = "Today is Sunday"
print(msg)
>>> Today is Sunday

# 打印格式化字符串
print("I want to have {} for the dinner, and {} for the breakfast".format("chicken", "cake"))
>>> I want to have chicken for the dinner, and cake for the breakfast
```

- 打印变量
```
a = 100
print(a)
>>> 100

b = [1, 2, 3]
print(b)
>>> [1, 2, 3]

c = ("A", "B", "C")
print(c)
>>> ("A", "B", "C")
```

# 条件语句
- 条件分支
```
score = int(input("What's your score?\n"))
if score >= 90:
  print("You're awesome!")
elif score >= 60:
  print("You need to work harder")
else:
  print("You suck...")
```

- 条件组合
```
color = "red"
size = 100
if color === "red" and size >= 90:
  print("它又大又红")
  
if color == "red" or size > 90:
  print("它要么大要么红")
  
if color != "red" and size >= 90:
  print("它很大, 但不是红色的")
  
if color != "red" or size < 90:
  print("它要么不大, 要么不红")
```

# 循环
- 循环固定次数
```
for a in range(5):
  print(a)
>>> 0
>>> 1
>>> 2
>>> 3
>>> 4
```

- 从最小数开始循环
```
for a in range(5, 10):
  print(a)
>>> 5
>>> 6
>>> 7
>>> 8
>>> 9
```

- 中断循环
```
names = ["mimi", "didi", "lili", "xixi", "coco", "dodo", "lolo"]
for name in names:
  if name == "coco":
    print("coco is found!")
    break
```

# tuple
只能保存固定数量的元素, 元素被初始化后不能修改.不能添加或者删除元素.
```
# 初始化tuple
names = ("mimi", "didi", "lili")

# 获取长度
num_names = len(names)
print("I have {} names".format(num_names))
>>> I have 3 names

# 遍历元素
for name in names:
  print(name)
>>> mimi
>>> didi
>>> lili

# 获取第一个元素
print("The first name is {}".format(names[0]))
>>> The first name is mini

# 获取最后一个元素（长度减一作为下标）
print("The last name is {}".format(names[len(names) - 1]))
>>> The last name is lili
```

# list
多个元素的集合, 可以任意添加, 删除元素.元素可以被修改.

- 初始化
```
scores = [70, 100, 90, 30]
print(scores)
>>> [70, 100, 90, 30]

# 获取长度
num_scores = len(scores)
print("I have {} scores".format(num_scores))
>>> I have 4 scores

# 获取第一个元素
print("The first score is {}".format(scores[0]))
>>> The first score is 70

# 获取最后一个元素（长度减一作为下标）
print("The last score is {}".format(scores[len(scores) - 1]))
>>> The last score is 30
```

- 遍历
```
scores = [70, 100, 90, 30]
for score in scores:
  print(score)
>>> 70
>>> 100
>>> 90
>>> 30
```

- 添加删除
```
# 末尾添加
scores = [70, 100, 90, 30]
scores.append(59)
print(scores)
>>> [70, 100, 90, 30, 59]

# 中间插入
scores.insert(1, 200)
print(scores)
>>> [70, 200, 100, 90, 30, 59]

# 删除元素
scores.pop(1)
>>> 200
print(scores)
>>> [70, 100, 90, 30, 59]
```

- 修改元素
```
scores = [70, 100, 90, 30]
scores[1] = 129
print(scores)
>>> [70, 129, 90, 30]
```

- 测试是否包含某个元素
```
scores = [70, 100, 90, 30]
if 40 in scores:
  print("40 is in the scores!")
```

- 排序和反转
```
# 反转
scores = [70, 100, 90, 30]
scores.reverse()
print(scores)
>>> [30, 90, 100, 70]

# 排序（从小到大）
scores.sort()
print(scores)
>>> [30, 70, 90, 100]
```

# 字典
建立从key到value的映射, 可以从key找到value, 但是无法从value找到key.

字典变量名尽量使用key_to_value的形式.

- 初始化
```
# 建立一个空字典
name_to_score = {}

# 建立一个非空字典
name_to_score = { "mimi": 100, "didi": 200, "lili": 300 }

# 获取长度
num_scores = len(name_to_score)
print(num_scores)
>>> 3
```

- 添加元素
```
name_to_score = {}
name_to_score["mini"] = 100
print(name_to_score)
>>> {"mini": 100}

name_to_score["didi"] = 200
print(name_to_score)
>>> {"mini": 100, "didi": 200}
```

- 更新value
```
name_to_score = { "mimi": 100}
print(name_to_score)
>>> {"mini": 100}

name_to_score["mimi"] = 200
print(name_to_score)
>>> {"mini": 200}
```

- 查找value
```
name_to_score = { "mimi": 100, "didi": 200, "lili": 300 }

name = "coco"
if name in name_to_score:
  score = name_to_score[name]
  print("I have a score for {}: {}".format(name, score))
else:
  print("There's no score for {}".format(name))
```

- 遍历
```
# 依次遍历key, value
name_to_score = { "mimi": 100, "didi": 200, "lili": 300 }
for name, score in name_to_score.items():
  print("key is {}, value is {}")
>>> key is mini, value is 100
>>> key is didi, value is 200
>>> key is lili, value is 300

# 获得所有的key, 并转成list
names = list(name_to_score.keys())
print(names)
>>> ["mini", "didi", "lili"]

# 获得所有的value, 并转成list
scores = list(name_to_score.values())
print(scores)
>>> [100, 200, 300]
```

- 删除元素
```
name_to_score = { "mimi": 100, "didi": 200, "lili": 300 }
del name_to_score["mini"]
print(name_to_score)
>>> { "didi": 200, "lili": 300 }
```

# 函数
