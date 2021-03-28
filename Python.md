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
# 循环
# tuple
# list
# 字典
# 函数
