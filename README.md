# ioflow
理想情况下，模型本身和模型如何训练，如何存储保存的模型等应该通过隔离开，这样就可以达到彼此解耦分工合作的目的。

## 对外提供接口
理想情况下，模型程序以 docker 镜像的形式提供，这样可以最大化解决依赖问题。
```bash
docker run -v path/to/configure_file.yaml:/configure_file.yaml some_model_image:label
```
其中：
* some_model_image 是对应算法的 docker 镜像
* label （例如 0.1, test.bilstm_crf 等），方便指定特定的版本或者算法变体等。
* path/to/configure_file.yaml 是包含所有配置信息的文件，配置信息包含：
    * 语料
    * 训练状态收集服务
    * 模型上传
    * 模型性能收集


## 对内接口
模型的运行应该和平台架构隔离的，因此使用 SDK 的方式构建接口抽象；

理想情况下的模型程序应该类似于：
```python
from ioflow.corpus import Corpus
from ioflow.task_status import TaskStatus
from ioflow.model_saver import ModelSaver
from ioflow.performance_metrics import PerformanceMetrics
from ioflow.configure import read_configure

from utils import build_input_func
from model import Model

raw_config = read_configure()
model = Model(raw_config)

config = model.get_default_config()
config.update(raw_config)

task_status = TaskStatus(config)

# read data according configure
corpus = Corpus(config)
corpus.prepare()
train_data_generator_func = corpus.get_generator_func(corpus.TRAIN)
eval_data_generator_func = corpus.get_generator_func(corpus.EVAL)

# send START status to monitor system
task_status.send_status(task_status.START)

# train and evaluate model
train_input_func = build_input_func(train_data_generator_func, config)
eval_input_func = build_input_func(eval_data_generator_func, config)

evaluate_result, export_results, final_saved_model = model.train_and_eval_then_save(
    train_input_func,
    eval_input_func,
    config
)

task_status.send_status(task_status.DONE)

if evaluate_result:
    performance_metrics = PerformanceMetrics(config)
    performance_metrics.set_metrics('test_loss', evaluate_result['loss'])
    performance_metrics.set_metrics('test_acc', evaluate_result['acc'])

model_saver = ModelSaver(config)
model_saver.save_model(final_saved_model)
````

