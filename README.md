# 基于Mlp-Mixer的物理层信道预测


## 1.问题描述
* PMI流程：每个时隙根据当前时隙及历史时刻的信道数据，估计下一时隙的PMI索引，该PMI值经反馈时延后，在下一个时隙使用；
* 输入信息：1、每个时隙的信道数据H，对应维度为(16RB, 4Rx, 16Tx)；2、每个时隙各个PMI（1024个）对应的度量值SI；
* 预测内容：下一个时隙的PMI索引，表示成4维向量(i4,i3,i2,i1)；每个元素取值均为正整数，最大值分别为[2,4,8,16]；
* 目的：预测的PMI尽可能匹配下一时隙，即为使预测PMI在下一时隙的度量值SI最大化；

## 2.数据集说明
* 信道数据：H_data_all.npy，数据维度(M, 200T, 16RB, 4Rx, 16Tx)，分别表示总共M个大场景，每个场景包含连续200个时刻的信道数据；
* PMI-互信息数据：SI_data_all.npy，与信道数据对应，每个时隙所有PMI（1024个）对应的互信息，可表示为SI(H_t,PMI)。
* 所有数据分为两类，以下简称为CDL和TDL数据集：

```bash
  CDL数据集：H_data_all_CDL.npy, SI_data_all_CDL.npy;
  TDL数据集：H_data_all_TDL.npy, SI_data_all_TDL.npy;
```
* CDL和TDL数据集的数据维度相同，总共720个大场景，每个场景包含连续200个时刻的信道数据。细分720个大场景，可以变形为[3, 3, 5, 16]:
```bash
    3：A/B/D三类大场景
    3：时延类型 300 30 900
    5：速度类型 15 30 3 45 60
    16：信噪比 0:15
```

## 3.生成训练数据
* 在TDL数据集上，生成利用历史5个时隙的信道数据，预测未来T+1时隙PMI的训练集和测试集：
```bash
python generate_data.py --scenes 720 --moments 200 --steps 1 --history_len 5 --channel_data dataset/H_data_all_TDL.npy --SI_data dataset/SI_data_all_TDL.npy --train_data data/train_TDL.npz --test_data data/test_TDL.npz
```
* 基于其他数据集(请保持与CDL/TDL相似的数据格式），生成利用历史m个时隙的信道数据，预测未来T+n时隙PMI的训练集和测试集：
```bash
python generate_data.py --scenes {scenes} --moments {moments} --steps {n} --history_len {m} --channel_data {path of your channle data} --SI_data {path of your SI data} --train_data {path of your generated train data} --test_data {path of your generated test data}
```

## 4.训练模型
* 在TDL生成的数据集上，使用默认参数训练模型：
```bash
python train.py --SI_data dataset/SI_data_all_TDL.npy --train_data data/train_TDL.npz --test_data data/test_TDL.npz --save_cp ckpt/TDL.pth
```
* 继续训练/在预训练模型上fine-tune：
```bash
python train.py --SI_data dataset/SI_data_all_TDL.npy --train_data data/train_TDL.npz --test_data data/test_TDL.npz --save_cp ckpt/TDL.pth --pretrained True --pretrained_path {path of pretrained checkpoint} --lr 0.0003
```

## 5.针对细分场景，在测试集上测试结果
* TDL数据集上，在一维场景下的测试：
```bash
python inference.py --SI_data dataset/SI_data_all_TDL.npy  --test_data data/test_TDL.npz --pretrained True --pretrained_path ckpt/TDL.pth 
```
* TDL数据集上，在三维场景下的测试,最终结果保存在npy和txt文件中：
```bash
python everyscene.py --SI_data dataset/SI_data_all_TDL.npy  --test_data data/test_TDL.npz --pretrained True --pretrained_path ckpt/TDL.pth 
```
## 6.补充
### 数据集
* 可以使用除CDL/TDL外的其他数据集，但请注意数据的格式应当与CDL/TDL相同，对应维度应该为(M, T, 16RB, 4Rx, 16Tx)，当场景数和时刻数改变时，修改对应的scenes和moments参数。

### 参数设置
* 生成训练集测试集的参数：steps定义未来T+N时隙预测中N的大小，默认为1；history_len定义使用历史时隙的长度，默认为5。
* 模型参数：patch_size定义将数据分块的块的小大，默认为8，也可设置为2或4，相应的减小块的大小，模型参数量也会降低。
* 优化器参数：lr初始设置为0.003，可快速收敛，后续降低以提高性能；weight_decay大小对模型性能影响非常小，默认设置为1e-3。

### 训练策略
* 在简单的场景下如CDL，模型收敛较快，复杂场景下如TDL，模型收敛较慢。偏向于使用手动调学习率的策略：先使用0.003的学习率让模型收敛，再使用0.0003的学习率继续训练到收敛，最后使用0.00003的学习率训练1~5个epoch即可完成训练。也可以使用自动的学习率衰减策略，但经过测试，手动调学习率能够获得更好的效果。
* 在一张1080Ti的训练条件下，大约12h能够完成训练。
