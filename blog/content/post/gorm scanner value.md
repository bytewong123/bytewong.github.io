---
title: gorm scanner value
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["golang"]
tags: ["golang"]
---

# gorm scanner value
#技术/golang/代码风格


# gorm自动加解密字段
# 背景
ESOP系统的数据项包含了员工行权数量、授予数量、税率和税额等敏感信息，从这些数据出发，能够推演出期权数据和员工薪酬的规律。这些数据的安全等级非常高，在DB、日志中都绝对不能出现明文信息。针对数据的全流程加密，保证数据安全的同时，对于研发来讲也是一种保护。那么针对数据加密落库的方案，有什么优雅的解决方案吗？
 
在我们定义Model时，model里的每个字段都是要映射到数据库的每一个column上的。在gorm中，我们通过go标签的方式来指定model和数据库表的绑定关系。
```go
type Person struct {
    IDNumber  string `gorm:"column:id_number"`
}
```

 
那么在明确了绑定关系之后，数据库字段值和Model结构体的值之间是怎么转换的呢？
go源码中定义了两个关键的接口，Scanner和Valuer来实现不同类型之间的值转换。
```go
// Users/tuocheng/go/go1.16/src/database/sql/sql.go

// Scanner is an interface used by Scan.
type Scanner interface {
   // Scan assigns a value from a database driver.
   //
   // The src value will be of one of the following types:
   //
   //    int64
   //    float64
   //    bool
   //    []byte
   //    string
   //    time.Time
   //    nil - for NULL values
   //
   // An error should be returned if the value cannot be stored
   // without loss of information.
   //
   // Reference types such as []byte are only valid until the next call to Scan
   // and should not be retained. Their underlying memory is owned by the driver.
   // If retention is necessary, copy their values before the next call to Scan.
   Scan(src interface{}) error
}

// Users/tuocheng/go/go1.16/src/database/sql/driver/types.go

// Valuer is the interface providing the Value method.
//
// Types implementing Valuer interface are able to convert
// themselves to a driver Value.
type Valuer interface {
   // Value returns a driver Value.
   // Value must not panic.
   Value() (Value, error)
}
```
 
在convertAssignRows进行转换时，会根据原类型和目标类型来区分是否需要调用scanner.Scan方法来转换。
 
```go
//Users/tuocheng/go/go1.16/src/database/sql/convert.go

// convertAssignRows copies to dest the value in src, converting it if possible.
// An error is returned if the copy would result in loss of information.
// dest should be a pointer type. If rows is passed in, the rows will
// be used as the parent for any cursor values converted from a
// driver.Rows to a *Rows.
func convertAssignRows(dest, src interface{}, rows *Rows) error {
   // Common cases, without reflect.
   // string、int、time.Time等类型，不用反射直接转
   switch s := src.(type) {
   // 省略了 byte[]， decimalDecompose， nil类型对应的转换处理
   case string:
      switch d := dest.(type) {
      case *string:
         if d == nil {
            return errNilPtr
         }
         *d = s
         return nil
      case *[]byte:
         if d == nil {
            return errNilPtr
         }
         *d = []byte(s)
         return nil
      case *RawBytes:
         if d == nil {
            return errNilPtr
         }
         *d = append((*d)[:0], s…)
         return nil
      }
   case time.Time:
      switch d := dest.(type) {
      case *time.Time:
         *d = s
         return nil
      case *string:
         *d = s.Format(time.RFC3339Nano)
         return nil
      case *[]byte:
         if d == nil {
            return errNilPtr
         }
         *d = []byte(s.Format(time.RFC3339Nano))
         return nil
      case *RawBytes:
         if d == nil {
            return errNilPtr
         }
         *d = s.AppendFormat((*d)[:0], time.RFC3339Nano)
         return nil
      }
   }
   
   
   // 省略部分代码


   // 判断目标类型是否实现了scanner接口，若实现了该接口，便调用该接口进行转换
   if scanner, ok := dest.(Scanner); ok {
      return scanner.Scan(src)
   }

   // 未实现scanner接口，继续尝试用反射进行转换
   
   // 省略部分代码

   return fmt.Errorf(“unsupported Scan, storing driver.Value type %T into type %T”, src, dest)
}
```

 
**那么当我们自定义的类型实现了Scanner接口和Valuer接口，orm框架就能够为我们实现自动的“拆装箱”。**
 
# 解决方案
既然有了自动“拆装箱”的机制以及对应的scanner/valuer接口，我们便可以实现自己的“拆装箱”策略。我们在“拆箱”的时候进行加密入库，在“装箱”的时候进行解密反解析，便可以在model层就实现几乎无感的加解密。
 
最终我们包装了一个KmsString类，通过kms托管密钥来对字段值进行加解密。
```go
// KmsString 实现 gorm string 类型加解密，保存字符串密文到数据库
type KmsString string

// KmsTypeFlag .
func (KmsString) KmsTypeFlag() {}

func (s KmsString) Value() (driver.Value, error) {
   // encryptString 自定制的加密策略
   value, err := encryptString(string(s))
   if err != nil {
      return nil, err
   }
   return value, nil
}

func (s *KmsString) Scan(src interface{}) error {
   value, err := convertString(src)
   if err != nil {
      return err
   }
   if value != “” {
      // decryptString 自定制的解密策略
      str, err := decryptString(value)
      if err != nil {
         return err
      }
      *s = KmsString(str)
   }
   return nil
}
```
 
利用该类，我们可以直接将model中的字段与数据库表绑定。在gorm进行scan和value的同时，完成数据的加解密。
```go
type Person struct {
    IDNumber  KmsString `gorm:"column:id_number"`
}
```

 
 
# 效果及拓展
通过实现scanner/valuer，我们实现了在model层“拆装箱”的过程中自动加解密。上层service, biz层均无感知，实现了加解密逻辑的终极收敛。
除此之外，我们还可以利用scanner/valuer轻松的实现一些特别的功能。
1. 实现decimal类，在入库时乘以放大倍数存成bigint, 在读取时自动除以放大倍数转成decimal保证精度。 
2. 定义ExtraMap类，在入库时json.Marshal存成varchar, 在读取时自动unmarshal成map。收敛marshal/unmarshal的逻辑在一处。
