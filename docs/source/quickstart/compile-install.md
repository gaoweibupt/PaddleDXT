# 编译和安装

## 源码编译和安装

### 环境准备
PaddleDTX 使用 golang 进行开发，当您使用源码进行编译和安装时，首先需要准备源码以及编译的环境：

* 系统环境, 推荐使用Linux或MacOS操作系统
* golang编译器, 推荐版本1.13+
[点击下载安装golang](https://studygolang.com/dl)
* git源码下载工具
[点击下载安装git](https://git-scm.com/download)
* make编译工具
	- ubuntu:
	```
	$ sudo apt-get install -y make
	```
	- centos:
	```
	$ sudo yum install -y make
	```
	- macOS:
	通过iTunes下载安装xcode

### 源码编译

1. 下载源码
	```
	$ git clone https://github.com/PaddlePaddle/PaddleDTX.git
	```
2. 编译XuperDB
	```
	$ cd PaddleDTX/xdb
	$ make
	```
	编译产出为 output 文件夹，内容为：
	```
	├── conf
	│   ├── config-dataowner.toml
	│   └── config-storage.toml
	├── xdata                   //数据存储服务启动二进制,根据配置文件的不同有数据持有节点或存储节点两种类型
	└── xdata-cli               //数据存储服务操作客户端
	```
3. 编译Distributed AI
	```
	$ cd ../dai
	$ make
	```
	编译产出为 output 文件夹，内容为：
	```
	├── conf
	│   ├── config-cli.toml     //客户端配置,也可以使用config.toml作为客户端配置
	│   └── config.toml         //服务配置
	├── executor                //DAI任务执行节点服务启动二进制
	├── executor-cli            //DAI任务执行节点操作客户端
	└── requester-cli           //计算需求节点操作客户端
	```
4. 编译区块链合约
	```
	$ go build -o paddlempc blockchain/xchain/contract/main.go
	```
	编译产出为 paddlempc 合约文件，为安装在xchain 区块链上的合约文件。

### 网络部署

1. 部署区块链网络

	PaddleDTX 使用区块链网络支撑计算层和去中心化存储网络，底层依赖可以使用同一个区块链网络。
	<br>
	这里我们使用的百度超级链 xchain v3.9 作为底层区块链网络，可参考 [XuperChain环境部署](https://xuperchain.readthedocs.io/zh/latest/quickstart.html) 来搭建区块链网络。
	<br>
	您需要了解如何创建合约账户、部署智能合约，详细参考 [部署 native 合约](https://xuperchain.readthedocs.io/zh/latest/advanced_usage/create_contracts.html) ，注意这里我们使用的是 golang native 合，需要修改xchain.yaml来支持 native 合约。
	<br>
	合约安装过程如下：
	```
	# 定义合约账户和合约名称
	$ export contractAccount='1234567890123456'
	$ export contractName='paddlempc'

	# 创建区块链账户, 并转 token
	$ ./xchain-cli account newkeys --strength 1 -o ukeys
	$ export address=`cat ./ukeys/address`
	$ ./xchain-cli transfer --to $address --amount 1000000000000

	# 创建区块链合约账户, 并转 token
	$ ./xchain-cli account new --account $contractAccount --fee 1000 --keys ./ukeys
	
	# 给合约账户转 token
	$ ./xchain-cli transfer --to XC${contractAccount}@xuper --amount 100000000000 --keys ./ukeys
	
	# 安装合约
	$ ./xchain-cli native deploy --account XC${contractAccount}@xuper --runtime go -a '{"creator":"XC${contractAccount}@xuper"}' --cname $contractName ./$contractName --fee 19267894 --keys ./ukeys

	# 查询合约安装的状态
	$ ./xchain-cli contract query $contractName
	```
2. 部署 XuperDB
	
	每一个 XuperDB 的节点都有一对公私钥，用来标识节点的账户，公私钥可以通过如下XuperDB的客户端命令进行获取：
	```
	$ ./xdata-cli nodes genkey
	```
	请妥善保存您创建的公私钥对，在后续的配置及命令行使用时您将会频繁的用到它；如果您是第一次搭建网络的话，建议使用默认配置文件中的私钥。
	<br><br>
	XuperDB 包含两种类型的节点服务，数据持有节点和存储节点，我们需要分别启动这两种服务。为了方便，我们这里两种类型的服务分别启动一个。
	首先进入到 xdb 的编译产出 output 文件中：

	1. 启动数据存储节点

		修改配置文件，修改内容如下：
		```
		# vim conf/config-storage.toml
		# 
		listenAddress = ":8122"
		publicAddress = "127.0.0.1:8122"

		# genkey创建的私钥, 对账户使用不熟悉的话建议使用默认账户
		privateKey = "5572e2fa0c259fe798e5580884359a4a6ac938cfff62d027b90f2bac3eceef79"

		[storage.blockchain]
		    [storage.blockchain.xchain]
		        mnemonic = "充 雄 孔 坝 低 狠 争 短 摸 拜 晨 造"
		        contractName = "paddlempc"
		        contractAccount = "XC1234567890123456@xuper"
		        chainAddress = "127.0.0.1:37101"
		        chainName = "xuper"
		```
		其中，listenAddress和publicAddress 指定服务监听的地址及对外暴露的地址，blockchain配置中使用区块链网络部署时创建的账户助记词、合约账户及合约名。

		启动服务：
		```
		$ nohup ./xdata -c conf/config-storage.toml > storage.log &
		```

	2. 启动数据持有节点

		修改配置文件，修改内容如下：
		```
		# vim conf/config-dataowner.toml
		# 
		listenAddress = ":8123"
		publicAddress = "127.0.0.1:8123"

		# genkey创建的私钥, 对账户使用不熟悉的话建议使用默认账户
		privateKey = "5572e2fa0c259fe798e5580884359a4a6ac938cfff62d027b90f2bac3eceef79"

		[storage.blockchain]
		    [storage.blockchain.xchain]
		        mnemonic = "充 雄 孔 坝 低 狠 争 短 摸 拜 晨 造"
		        contractName = "paddlempc"
		        contractAccount = "XC1234567890123456@xuper"
		        chainAddress = "127.0.0.1:37101"
		        chainName = "xuper"
		[storage.mode]
			[storage.mode.local]
				rootPath = "./data/slices"
		```
		其中，listenAddress和publicAddress 指定服务监听的地址及对外暴露的地址，blockchain配置中使用区块链网络部署时创建的账户助记词、合约账户及合约名，rootPath指定文件存储的本地路径。

		启动服务：
		```
		$ nohup ./xdata -c conf/config-dataowner.toml > dataowner.log &
		```

		*注意：一般构建 PaddleDTX 网络至少需要两方参与，即两个数据持有节点，对应了两个计算任务执行节点，这里为了说明方便启动一个数据持有节点，您也可以根据实际需求自行启动多个数据存储节点和数据持有节点；配置中的privateKey参数为节点的身份，不同privateKey即对应了不同的身份。*

	3. 查看服务状态

		使用 xdata-cli 客户端执行如下命令，请求数据持有节点查看存储节点的在线状态：
		```
		$ ./xdata-cli nodes list --host http://127.0.0.1:8123
		```
3. 部署Distributed AI

	一般多方安全计算至少由两个任务执行节点，所以这里部署两个任务执行节点。

	*注意: 任务执行节点的账户也是通过公私钥对来标明, 通常我们一个任务执行节点对应了一个数据持有节点, 这种情况下任务执行节点和对应的数据持有节点需要使用相同的公私钥对。*

	1. 准备两个任务执行节点的配置
		```
		$ cd PaddleDTX/dai/
		$ cp -r output executor1
		$ cp -r output executor2
		```
		
		需要分别修改对应的conf/config.toml文件：
		
		
		```
		# executor1
		listenAddress = ":8184"
		publicAddress = "127.0.0.1:8184"
		privateKey = "5572e2fa0c259fe798e5580884359a4a6ac938cfff62d027b90f2bac3eceef79"
		[executor.storage]
			[executor.storage.XuperDB]
				host = "http://127.0.0.1:8123"
		[executor.blockchain]
		    [executor.blockchain.xchain]
		        mnemonic = "充 雄 孔 坝 低 狠 争 短 摸 拜 晨 造"
		        contractName = "paddlempc"
		        contractAccount = "XC1234567890123456@xuper"
		        chainAddress = "127.0.0.1:37101"
		        chainName = "xuper"
		```

		```
		# executor2
		listenAddress = ":8185"
		publicAddress = "127.0.0.1:8185"
		privateKey = "5572e2fa0c259fe798e5580884359a4a6ac938cfff62d027b90f2bac3eceef79"
		[executor.storage]
			[executor.storage.XuperDB]
				host = "http://127.0.0.1:8123"
		[executor.blockchain]
		    [executor.blockchain.xchain]
		        mnemonic = "充 雄 孔 坝 低 狠 争 短 摸 拜 晨 造"
		        contractName = "paddlempc"
		        contractAccount = "XC1234567890123456@xuper"
		        chainAddress = "127.0.0.1:37101"
		        chainName = "xuper"
		```

	2. 启动服务

		分别在对应文件夹下执行如下命令，启动任务执行节点：
		```
		$ nohup ./executor &
		```

		通过命令查看有两个 executor 的进程，则启动成功：
		```
		$ ps -aux | grep executor
		```



## 通过 docker 安装

### 环境准备
PaddleDTX也支持使用 docker 进行编译、安装和使用，您需要准备如下环境：
* docker, 推荐版本18.03+ [点击下载安装 docker](https://docs.docker.com/get-docker/)
* docker-compose, 推荐版本1.26.0+ [点击下载安装 docker-compose](https://github.com/docker/compose/releases)

### 制作镜像

1. 制作XuperDB镜像
	```
	$ cd PaddleDTX/xdb
	$ sh build_image.sh
	```
	产出镜像名和版本号为 *registry.baidubce.com/paddledtx/paddledtx-dai:1.0* ，可以通过修改 *build_image.sh* 脚本来修改镜像名和版本号。


2. 制作 Distributed AI 镜像
	```
	$ cd PaddleDTX/dai
	$ sh build_image.sh
	```
	产出为计算需求节点和任务执行节点两个镜像, 镜像名和版本号分别为：
	* registry.baidubce.com/paddledtx/xdb-storage:1.0
	* registry.baidubce.com/paddledtx/xdb-dataowner:1.0
	实际上他们是使用不同镜像名的同一个镜像，可以通过修改 *build_image.sh* 脚本来修改镜像名和版本号。

3. 编译合约
	```
	$ export contractName='paddlempc'
	$ docker run -it --rm \
      -v $(dirname ${PWD}):/workspace \
      -v ~/.ssh:/root/.ssh \
      -w /workspace \
      -e GONOSUMDB=* \
      -e GO111MODULE=on \
      golang:1.13.4 sh -c "cd dai && go build -o ../testdata/blockchain/contract/$contractName ./blockchain/xchain/contract"
	```

### 网络部署

1. 部署区块链网络
	```
	$ cd PaddleDTX/testdata/blockchain
	$ docker-compose -f docker-compose.yml up -d
	```
	搭建了三个节点的区块链网络，对应的区块链相关配置在文件夹 blockchain/xchain1/conf、blockchain/xchain2/conf、blockchain/xchain3/conf 下，需要调整配置时在网络拉起前进行修改。

	可以通过容器中的 xchain-cli客户端进行区块链上的一些操作，例如创建合约账户及安装智能合约。
	

	```
	# 定义合约账户和合约名称
	$ export contractAccount='1234567890123456'
	$ export contractName='paddlempc'

	# 创建区块链账户, 并转 token
	# 这里也可以使用已有的区块链账户, 文章后面的修改区块链配置可以不操作
	$ docker exec -it xchain1.node.com sh -c "./xchain-cli account newkeys --strength 1  -o user"
	$ export address=`cat user/address`
	$ docker exec -it xchain1.node.com sh -c "./xchain-cli transfer --to $address --amount 1000000000000"
	
	# 创建合约账户
	$ docker exec -it xchain1.node.com sh -c "./xchain-cli account new --account $contractAccount --fee 1000 --keys ./user"

	# 给合约账户转 token
	$ docker exec -it xchain1.node.com sh -c "./xchain-cli transfer --to XC${contractAccount}@xuper --amount 100000000000 --keys ./user"

	# 将合约拷贝到容器中
	$ docker cp ./contract/$contractName xchain1.node.com:/home/work/xchain/$contractName

	# 安装合约
	$ docker exec -it xchain1.node.com sh -c "./xchain-cli native deploy --account XC${contractAccount}@xuper --runtime go -a '{\"creator\":\"XC${contractAccount}@xuper\"}' --cname $contractName ./contract/$contractName --fee 19267894 --keys ./user --fee 19597986"

	# 查询合约安装的状态
	$ docker exec -it xchain1.node.com sh -c "./xchain-cli contract query $contractName"

	```

2. 部署 XuperDB

	数据持有节点将自己的隐私数据进行加密、切分、副本复制后分发到存储节点，存储节点是数据存储的物理节点。我们部署三个存储节点和两个数据持有节点, 两个数据节点模拟分别提供部分数据的两方。
	
	修改配置文件：
	```
	$ vim xdb/data1/conf/config.toml
	$ vim xdb/data2/conf/config.toml

	# 使用在区块链部署时创建的合约账户、合约以及助记词
	[dataOwner.blockchain]
    type = "xchain"
    [dataOwner.blockchain.xchain]
        mnemonic = "提 现 详 责 腐 贪 沉 回 涨 谓 献 即"
        contractName = "paddlempc"
        contractAccount = "XC1234567890123456@xuper"
        chainAddress = "xchain1.node.com:37101"
        chainName = "xuper"
	```

	```
	$ vim xdb/storage1/conf/config.toml
	$ vim xdb/storage2/conf/config.toml
	$ vim xdb/storage3/conf/config.toml

	# 使用在区块链部署时创建的合约账户、合约以及助记词
	[storage.blockchain]
    type = "xchain"
    [storage.blockchain.xchain]
        mnemonic = "提 现 详 责 腐 贪 沉 回 涨 谓 献 即"
        contractName = "paddlempc"
        contractAccount = "XC1234567890123456@xuper"
        chainAddress = "xchain1.node.com:37101"
        chainName = "xuper"
	```

	启动服务：
	```
	$ cd PaddleDTX/testdata/xdb
	$ docker-compose -f docker-compose.yml up -d
	```

	查看存储节点列表：
	```
	$ docker exec -it dataowner1.node.com sh -c "./xdata-cli nodes list --host http://dataowner1.node.com:80"
	```

3. 部署 Distributed AI

	部署两个任务执行节点，模拟由两方组成的多方安全计算网络，两个任务执行节点分别对应不同的数据持有节点。

	修改配置文件：
	```
	$ vim ./executor/node1/conf/config.toml
	$ vim ./executor/node2/conf/config.toml
	# 使用在区块链部署时创建的合约账户、合约以及助记词
	[executor.blockchain]
    type = 'xchain'
    [executor.blockchain.xchain]
        mnemonic = "提 现 详 责 腐 贪 沉 回 涨 谓 献 即"
        contractName = "paddlempc"
        contractAccount = "XC1234567890123456@xuper"
        chainAddress = "xchain1.node.com:37101"
        chainName = "xuper"
	```

	启动服务：
	```
	$ cd PaddleDTX/testdata/xdb
	$ docker-compose -f docker-compose.yml up -d
	```

