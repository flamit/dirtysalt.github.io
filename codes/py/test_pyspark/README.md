# NOTES

这个example是非常简单的case, 没有涉及virtualenv的部署，以及部署到yarn集群的情况。
真实情况远比这个要复杂得多。而且你大概率是跑在aws emr上的，所以emr的问题也需要考虑到。


# run on aws emr

## 打包venv成为venv.zip

venv.zip里面主要包含python可执行程序和不太变动的依赖

```bash
echo "make venv.zip"; cd venv; zip -rq ../venv.zip *; cd ..
aws s3 cp venv.zip s3://bucket/venv.zip
```

## 打包app成为app.zip

app.zip里面是逻辑需要的模块等等. 其中main.py是入口文件

```bash
echo "make app.zip"; zip -rq app.zip share
aws s3 cp app.zip s3://bucket/app.zip
aws s3 cp main.py s3://bucket/main.py
```

## 使用aws emr命令提交

```bash
aws emr add-steps --cluster-id <cluster-id> --steps Type=spark,Name=ParquetConversion,
Args=[--deploy-mode,cluster,--master,yarn,--conf,spark.yarn.submit.waitAppCompletion=true,
--conf,spark.yarn.appMasterEnv.PYSPARK_PYTHON=./venv/bin/python,
--conf,spark.yarn.appMasterEnv.PYSPARK_DRIVER_PYTHON=./venv/bin/python,
--archives,s3a://bucket/venv.zip#venv,
--py-files,s3a://bucket/app.zip,
s3a://bucket/main.py],ActionOnFailure=CONTINUE
```

其中 `archives` 里面指定的文件会在本地解压缩，`#` 后面部分的解压缩的目录名称. 我的理解是
`PYSPARK_PYTHON`是executor上面的python二进制， `PYSPARK_DRIVE_PYTHON` 是driver上面
的python二进制
