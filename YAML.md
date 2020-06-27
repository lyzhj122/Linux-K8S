# YAML



```yaml
--- !clarkevans.com/^invoice
invoice: 34843
date   : 2001-01-23
male: true
birthday: 1992-10-10 14:12:23  #iso 8601
flaws: null
bill-to: &id001      
   given  : Chris
   family : Dumars
   address:
      lines: |
            458 Walkman Dr.
            Suite #292
      city    : Royal Oak
      state   : MI
      postal  : 48046
ship-to: *id001
product:
    - sku         : BL394D
      quantity    : 4
      description : Basketball
      price       : 450.00
   - sku         : BL4438H
      quantity    : 1
      description : Super Hoop
      price       : 2392.00
tax  : 251.42
total: 4443.52
comments: >
    Late afternoon is best.
    Backup contact is Nancy
    Billsmer @ 338-4338.
```



```yaml
person:
  name: &name "Jack"    # &name 即把name当做一个变量，后面可以对他进行引用
  Occupation: 'Engineer'
  age: 32
  age: !!float 32   #强制类型转换为 float , 即：32.0
  gpa: 3.6
  gpa: !!str 3.6    #强制类型转换为 string , 即： "3.6"
  fav_num: 1e+10
  male: true
  birthday: 1989-10-29 15:23:12   #iso 8601
  flaws: null
#以上定义了person，可以进行存取： person.name, person.age ... 
  hobbies:
    - hiking
    - movies
    - riding bikes
  movie: ["Darking Knight", "Pie"]
  friends:
    - name: "Steph"
      age: 23
    - {name:"Mike", age: 23}
    - 
      name: "Joe"
      age: 22
  # 以上这三种对于friends的对象定义都是相同的 
description: >    # > 让yaml 把后面的文字进行render合并成一行
    There are two gRPC service 
    interfaces—ImageService and RuntimeService—that CRI
    container runtimes (or shims) must implement.
signature: |      # | 符号 表明下面的文字不进行合并，保持原有的分行格式
    Jack
    Hkeel
    email -  lyzhj122@gmail.com
id: *name   # 引用上面定义的 &name 变量， 即： "Jack"  
base: &base
  var1: value1
foo:
  <<: *base   #引用上面定义的 &base 变量，一个key:value 对，即： var1： value
  var2: value2
    
  
  
```

