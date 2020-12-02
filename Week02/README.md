### Golang error处理

#### 1.1 Golang 官方库对的error支持
  (1)Golang的错误比较轻量,Error的错误只需要实现buildin包下的error的interface即可
  ```
    type error interface {
      Error() string
    }
  ```
  (2)Goland的默认支持实现为errors包下的实现
  ```
    // errors.go文件
    
    package errors

    // New returns an error that formats as the given text.
    // Each call to New returns a distinct error value even if the text is identical.
    func New(text string) error {
      return &errorString{text}
    }

    // errorString is a trivial implementation of error.
    type errorString struct {
      s string
    }

    func (e *errorString) Error() string {
      return e.s
    }

  ```
  (3)1.3后支持了wrap支持，主要支持的Unwrap、Is和As方法。还使用%w来实现wrap功能，但是不包含堆栈信息
  ```
  // wrap.go
  
    func Unwrap(err error) error {
      u, ok := err.(interface {
        Unwrap() error
      })
      if !ok {
        return nil
      }
      return u.Unwrap()
    }

    func Is(err, target error) bool {
      if target == nil {
        return err == target
      }

      isComparable := reflectlite.TypeOf(target).Comparable()
      for {
        if isComparable && err == target {
          return true
        }
        if x, ok := err.(interface{ Is(error) bool }); ok && x.Is(target) {
          return true
        }
        // TODO: consider supporting target.Is(err). This would allow
        // user-definable predicates, but also may allow for coping with sloppy
        // APIs, thereby making it easier to get away with them.
        if err = Unwrap(err); err == nil {
          return false
        }
      }
    }

    func As(err error, target interface{}) bool {
      if target == nil {
        panic("errors: target cannot be nil")
      }
      val := reflectlite.ValueOf(target)
      typ := val.Type()
      if typ.Kind() != reflectlite.Ptr || val.IsNil() {
        panic("errors: target must be a non-nil pointer")
      }
      if e := typ.Elem(); e.Kind() != reflectlite.Interface && !e.Implements(errorType) {
        panic("errors: *target must be interface or implement error")
      }
      targetType := typ.Elem()
      for err != nil {
        if reflectlite.TypeOf(err).AssignableTo(targetType) {
          val.Elem().Set(reflectlite.ValueOf(err))
          return true
        }
        if x, ok := err.(interface{ As(interface{}) bool }); ok && x.As(target) {
          return true
        }
        err = Unwrap(err)
      }
      return false
    }

    var errorType = reflectlite.TypeOf((*error)(nil)).Elem()
  ```
  
#### 1.2 对golang官方库实现的扩展pkg/errors
  扩展errors支持：https://github.com/pkg/errors
  
  pkg/errors提供了对官方库error的包装，除了提供错误信息，还可以跟踪报错的堆栈信息
  
  

#### 1.3 golang的错误类型处理的姿势
  (1).Sentinel Error,预定义的特定错误，是指自己通过errors.New()方法创建一个定义的错误，例如：var Canceled = errors.New("context canceled")
  (2).Error type,自定错误类型，是指自己定义一个错误类型，实现error的接口
  ```
    type MyError struct {
      Msg string
      File string;
      Line int
    }
    
    func (e *MyError) Error() string {
      return fmt.Sprintf("%s:%d:%s",e.File,e.Line,e.Msg)
    }
    
    func test() error {
      return &MyError{"Something happened","server.go",42}
    }
  ```
  (3).Opaque errors,透明化的errors,是指对外只提供是否产生错误，而不提供具体错误内容信息,如下：
  ```
      type temporary interface {
        Tempporary() bool
      }

      func IsTemporary(err error) bool {
        te,ok := err.(temporary)
        return ok && te.Temporary()
      }
  ```
  对外提供IsTemporary()方法检测是否是某个错误，而不对外提供具体的信息
  
  对于三种类型的使用

#### 1.4 golang错误处理的实践
  1.在业务系统应用程序中，如果是自己写的函数返回错误，使用errors.New或者errors.Errorf返回错误(保存有堆栈信息)

  2.在业务系统应用程序中，如果是调的该业务系统的其他包的或者项目返回的错误，错误直接返回(不用进行包装)，比如service层调用dao

  3.在业务系统应用程序中，如果是调用的第三方库，标准库或者调用自己公司的基础库，要使用Wrap或者Wrapf进行包装根因(最底层的错误信息)

  4.在程序的顶部，或者工作的goroutine顶部打印日志，使用%+v来记录堆栈信息

  5.顶层再使用errors.Cause获取root error(根因)，再进行和sentinel error的判断

  6.在业务系统应用程序中，对外使用error code(统一错误码)返回给外部


#### 1.5 课后作业
```

    var (
    	ErrDataNotFound = errors.New("record not found")
    )

    type User struct {
        Id uint64 `json:"Id"`
        Name string `json:"name"`
        Age  int32  `json:"age"`
    }

    type Dao interface {
        Get(id uint64) interface{}
        List() interface{}
        Create() 
        Update()
        Delete(id uint64) 
    }

    type UserDao struct {}
    
    // Dao层获取到底层错误，使用errors的Wrap进行包装
    fuc(user *UserDao) Get(id uint64) (*User, error) {
        user := User{}
        err := db.Where("id = ?",id).Find(user).Error
        
        if errors.Is(err,sql.ErrNoRows){
            retrun errors.Wrap(err,fmt.Sprintf("find user null,user id: %v",id))
        }
        return &user,nil
    }

    // 业务层获取到错误直接往上层抛
    type UserService struct {}
    
    func (s *Service) FindUserByID(userID int) (*model.User, error) {
        return dao.FindUserByID(userID)
    }
    
    
```
