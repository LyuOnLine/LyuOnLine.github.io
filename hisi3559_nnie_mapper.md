Hi3559v200只支持Caffe-1.0框架模型，并需要使用nnie_mapper转换成硬件使用的wk格式模型格式。

本文主要内容:
- [nnie_mapper install and usage](#nniemapper-install-and-usage)


# nnie_mapper install and usage
  nnie_mapper依赖项比较多。在手册中给出的安装环境是: ubuntu 14.04, gcc 4.8.5, protobuf v3.6.1, opencv 3.4.2, CUDA 8.0，需要一一安装。
  为避免环境不一致引入的问题，可使用docker运行， dockerfile已经上传至[github](https://github.com/LyuOnLine/dockerfile/tree/master/Hisi/nnie_mapper)。

- docker编译:
    ```
    git clone https://github.com/LyuOnLine/dockerfile/
    cd dockerfile/Hisi/nnie_mapper
    docker build -t hisi/nnie_mapper .
    ```
- docker运行:
   ```
   # 拷贝docker_nnie_mapper.sh至PATH路径内
   cp dockerfile/Hisi/nnie_mapper/docker_nnie_mapper.sh /usr/local/bin/

   # 测试sdk中model转换
   cd HiSVP_PC_V1.2.2.5/software/data
   docker_nnie_mapper detection/yolov3/yolov3_func.cfg
   ```

