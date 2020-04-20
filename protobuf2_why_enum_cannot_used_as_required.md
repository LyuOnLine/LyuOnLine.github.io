**Why enum can't set required in protobuf2?**
----------------------------------------------
  
在使用Protobuf2时，应用enum类型，有两条原则。虽然在官方manual中，没有明确指出。但实践表明，最好遵守以免入坑...
- 原则一: 必须把"0"定义成UNKOWN，且必须放enum第一项。
- 原则二: enum用做struct的field时，必须是optional的，不能是required。

下面，通过具体的demo，看一下，如果不遵守这两个原则，会发生什么...

- 假设，proto定义如下:
    ```
    syntax = "proto2";
    package test;

    enum TestEnum {
        UNKOWN = 0;
        ENUM1 = 1;
        ENUM2 = 2;
    }

    message TestMessage {
        required TestEnum testEnum = 1;
        required int32 testInt = 2;
    }
    ```
    - 代码可以[github](https://github.com/LyuOnLine/test_examples/tree/master/protobuf/proto2.enum)下载

### 场景一: 正常使用时情况
- 分别生成go,python,c++版本的测试程序，对正常范围enum编码、解码，结果正常。
    - 输入testEnum=ENUM1, testInt=100时：
    - go,python,c++均正常编码。
    - proto编码为:   0x08,0x01,0x10,0x14

- go版本测试程序:

    ```go
    package main

    import (
        "fmt"
        "io/ioutil"
        pb "test"
        "github.com/golang/protobuf/proto"
    )

    func main() {
        var testEnum pb.TestEnum = 1
        var testInt int32 = 100
        msg := pb.TestMessage{TestEnum: &testEnum, TestInt: &testInt}
        buf, err := proto.Marshal(&msg)
        if err != nil {
            fmt.Printf("[ERROR] marshal failed! %v\n", err)
        }

        err = ioutil.WriteFile("/tmp/proto.dat", buf, 0644)
        if err != nil {
            fmt.Println("[ERROR] write to proto.dat failed!")
        }

        msg2 := &pb.TestMessage{}
        err = proto.Unmarshal(buf, msg2)
        if err != nil {
            fmt.Printf("[ERROR] unmarshal failed! %v\n", err)
        }
    fmt.Printf("message = %v\n", msg2)
    }
    ```
- c++版本
    ```
    #include <stdio.h>
    #include <string>
    #include "test.pb.h"
    using namespace test;
    using namespace std;

    int main(int argc, char *argv[]) {
        TestMessage msg;
        string buf;
        msg.set_testenum(TestEnum(1));
        msg.set_testint(20);
        msg.SerializeToString(&buf);
        
        FILE * fl = fopen("/tmp/proto.dat", "wb+");
        if(fl == NULL) {
            printf("open proto.dat failed!\n");
            return -1;
        }
        fwrite(buf.c_str(), 1, buf.length(), fl);
        fclose(fl);
        
        TestMessage msg2;
        msg2.ParseFromString(buf);
        printf("message: enum=%d, int=%d\n", msg2.testenum(), msg2.testint());
        return 0;
    }
    ```
- python版本测试程序:
    ```python
    import test_pb2

    if __name__ == "__main__":
        msg = test_pb2.TestMessage()
        msg.testEnum = 1
        msg.testInt = 100
        buf = msg.SerializeToString()
        
        with open("/tmp/proto.dat", "wb+") as fl:
            fl.write(buf)

        msg2 = test_pb2.TestMessage()
        msg2.ParseFromString(buf)
        print(msg2)
    ```
### 场景二： enum超界时情况
- 假设，使用中对enum赋与超界值100，三种语言行为是有差异的。
- cpp会触发assert failed，导致程序异常退出。
    ```
    Assertion failed: (::test::TestEnum_IsValid(value)), function _internal_set_testenum, file ./test.pb.h, line 281
    ```
- python会触发ValueError exception:
    ```
    ValueError: Unknown enum value: 100
    ```
- go不报错，且超界值正常参与编码。
    - 编码结果为: 0x08,0x64,0x10,0x64

### 场景三:  proto编码的二进制使用超界的enum，使用不同语言进行解码。
- cpp:
    - 会报错。
    - 把超界enum，解析成unkown。
    ```
    [libprotobuf ERROR google/protobuf/message_lite.cc:123] Can't parse message of type "test.TestMessage" because it is missing required fields: testEnum
    message: enum=0, int=20
    ```
- python:
    - 不会报错。
    - 把超界enum，解析成0。（默认不会打印)
- go:
    - 不会报错。
    - 超界的enum，解析成超界值本身。
    ```
    go run src/marshal.go
    message = testEnum:100 testInt:100
    ```
### 场景四: 修改proto，把enum改为optional.
- 场景二，实验结果不变。
    - cpp/python，越界值仍然会触发异常。
    - go仍然正常编码。
- 场景三，实验结果发生变化：
    - cpp版本不再报错，且越界值解析成0.


### 总结：
- 对Enum赋与越界值：
    - python/c++会触发异常。
    - 但go，由于enum本身是int的别名，不会做任何检查。
- 包含越界Enum值binary解析时，且enum为required类型时
    - c++版本，会报错，但正常解析为unkown。
    - android(java)版本，会触发异常，退出应用。
    - python版本，不会报错。解析为unkown。
    - go版本，不会报错，解析为越界值本身。
- 包含越界Enum值binary解析，且enum为optional类型时：
    - c++/java/python:均不报错，正常解析为Unkown。
    - go版本，不会报错，解析为越界值本身。

- 需要注意的是，Enum越界情况，并不是一定可以避免的。
    - 场景1: 升级协议时，比如业务需要增加一个新的ENUM_NEW。而server/client有可能由于升级不同步，marshal按新协议(testEnum=ENUM_NEW)，而unmarshal按老协议的情况。
    - 场景2: 程序或数据库存贮中，导致异常值，可能会超限。

        ```
        enum TestEnum {
            UNKOWN = 0;
            ENUM1 = 1;
            ENUM2 = 2;
            ENUM_NEW = 10;
        }
        ```
- 而Enum越界时，proto解析为unkown。但与required强类型检查，在部分语言中是冲突的。会导致c++报错，java中异常退出。
   而这种异常是我们不希望出现的.....