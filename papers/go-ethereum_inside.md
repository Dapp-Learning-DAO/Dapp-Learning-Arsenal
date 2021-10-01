# go-ethereum 源码

## 区别两种调用方式

### 裸调用 `call`

``` js
bytes memory input = abi.encodeWithSelector(Instance.Func2.selector);
(bool success, bytes memory data) = ins.call(input);
```

编译时无法提前知道返回数据长度，因此`CALL` 最后一个参数为 `0x00`

会再通过 `RETURNDATASIZE` 和 `RETURNDATACOPy` 将返回数据复制到 `data` 中

注意：节点会缓存上次返回的结果到 `in.returnData`，供后面查询

``` go
// go-ethereum/core/vm/instructions.go

func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
	for {
		// execute the operation
		res, err = operation.execute(&pc, in, callContext)
		// if the operation clears the return data (e.g. it has returning data)
		// set the last return to the result of the operation.
		if operation.returns {
			in.returnData = res
		}
	}
	return nil, nil
}

func opReturnDataSize(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
	scope.Stack.push(new(uint256.Int).SetUint64(uint64(len(interpreter.returnData))))
	return nil, nil
}

func opReturnDataCopy(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
	var (
		memOffset  = scope.Stack.pop()
		dataOffset = scope.Stack.pop()
		length     = scope.Stack.pop()
	)

	scope.Memory.Set(memOffset.Uint64(), length.Uint64(), interpreter.returnData[offset64:end64])
	return nil, nil
}
```

### 函数调用

``` js
(bool ret0, bool ret1) = instance.Func2();
```

编译时根据返回类型为 `(bool, bool)`，可以提前知道返回数据长度

因此 `CALL` 最后一个参数直接为 `0x40`

### 测试

``` js
// SPDX-License-Identifier: GPL-3.0

pragma solidity ^0.8.0;

contract Instance {
    function Func0() public {

    }

    function Func1() public pure returns (bool) {
        return true;
    }

    function Func2() public pure returns (bool, bool) {
        return (true, true);
    }
}

contract CallReturnDataLength {
    address public ins;

    constructor() {
        ins = address(new Instance());
    }

    function callFunc0() public {
        bytes memory input = abi.encodeWithSelector(Instance.Func0.selector);
        (bool success, bytes memory data) = ins.call(input);
        require(
            success && (data.length == 0 || abi.decode(data, (bool))),
            "TransferHelper: TRANSFER_FAILED"
        );
    }

    function callFunc1() public {
        bytes memory input = abi.encodeWithSelector(Instance.Func1.selector);
        (bool success, bytes memory data) = ins.call(input);
        require(
            success && (data.length == 0 || abi.decode(data, (bool))),
            "TransferHelper: TRANSFER_FAILED"
        );
    }

    function callFunc2() public {
        bytes memory input = abi.encodeWithSelector(Instance.Func2.selector);
        (bool success, bytes memory data) = ins.call(input);
        require(
            success && (data.length == 0 || abi.decode(data, (bool))),
            "TransferHelper: TRANSFER_FAILED"
        );
    }

    function directCallFunc0() public {
        Instance instance = Instance(ins);
        instance.Func0();
    }

    function directCallFunc1() public view {
        Instance instance = Instance(ins);
        bool ret0 = instance.Func1();
        require(ret0);
    }

    function directCallFunc2() public view {
        Instance instance = Instance(ins);
        (bool ret0, bool ret1) = instance.Func2();
        require(ret0 && ret1);
    }

    function directCallFunc2IgnoreReturn() public view {
        Instance instance = Instance(ins);
        instance.Func2();
    }
}

contract CallReturnDataLength2 {
    address public ins;

    constructor() {
        ins = address(new Instance());
    }

    function callFunc2() public {
        bytes memory input = abi.encodeWithSelector(Instance.Func2.selector);
        (bool success, bytes memory data) = ins.call(input);
        require(
            success && (data.length == 0 || abi.decode(data, (bool))),
            "TransferHelper: TRANSFER_FAILED"
        );
    }
}
```

#### callFunc0

``` json
[
	"0x2d692b",
	"0xf65fd7aa1eb820c80611b4c7c9b54c78f634b297",
	"0x0",
	"0xa4",
	"0x4",
	"0xa4",
	"0x0",
	"0xa8",
	"0xf65fd7aa1eb820c80611b4c7c9b54c78f634b297",
	"0x0",
	"0x0",
	"0x80",
	"0x9f",
	"0x43dd776c"
]
```

#### callFunc1

``` json
[
	"0x2d6900",
	"0xf65fd7aa1eb820c80611b4c7c9b54c78f634b297",
	"0x0",
	"0xa4",
	"0x4",
	"0xa4",
	"0x0",
	"0xa8",
	"0xf65fd7aa1eb820c80611b4c7c9b54c78f634b297",
	"0x0",
	"0x0",
	"0x80",
	"0xef",
	"0xe2c37ead"
]
```

#### callFunc2

``` json
[
	"0x2d692c",
	"0xf65fd7aa1eb820c80611b4c7c9b54c78f634b297",
	"0x0",
	"0xa4",
	"0x4",
	"0xa4",
	"0x0",
	"0xa8",
	"0xf65fd7aa1eb820c80611b4c7c9b54c78f634b297",
	"0x0",
	"0x0",
	"0x80",
	"0xdb",
	"0xd3ac2e67"
]
```

### directCallFunc0

``` json
[
	"0x2d60b7",
	"0xf65fd7aa1eb820c80611b4c7c9b54c78f634b297",
	"0x0",
	"0x80",
	"0x4",
	"0x80",
	"0x0",
	"0x84",
	"0x62c8d3f9",
	"0xf65fd7aa1eb820c80611b4c7c9b54c78f634b297",
	"0xf65fd7aa1eb820c80611b4c7c9b54c78f634b297",
	"0xc7",
	"0x87fbb464"
]
```

#### directCallFunc1

``` json
[
	"0x2d60cb",
	"0xf65fd7aa1eb820c80611b4c7c9b54c78f634b297",
	"0x0",
	"0x80",
	"0x4",
	"0x80",
	"0x20",
	"0x84",
	"0xcc551542",
	"0xf65fd7aa1eb820c80611b4c7c9b54c78f634b297",
	"0x0",
	"0xf65fd7aa1eb820c80611b4c7c9b54c78f634b297",
	"0xe5",
	"0xd4b798ba"
]
```

#### directCallFunc2

``` json
[
	"0x2d60f4",
	"0xf65fd7aa1eb820c80611b4c7c9b54c78f634b297",
	"0x0",
	"0x80",
	"0x4",
	"0x80",
	"0x40",
	"0x84",
	"0x42793c11",
	"0xf65fd7aa1eb820c80611b4c7c9b54c78f634b297",
	"0x0",
	"0x0",
	"0xf65fd7aa1eb820c80611b4c7c9b54c78f634b297",
	"0xd1",
	"0xc46ac896"
]
```

#### directCallFunc2IgnoreReturn

``` json
[
	"0x2d60f9",
	"0xf65fd7aa1eb820c80611b4c7c9b54c78f634b297",
	"0x0",
	"0x80",
	"0x4",
	"0x80",
	"0x40",
	"0x84",
	"0x42793c11",
	"0xf65fd7aa1eb820c80611b4c7c9b54c78f634b297",
	"0xf65fd7aa1eb820c80611b4c7c9b54c78f634b297",
	"0x95",
	"0x1bd10ac9"
]
```

## `opReturn` 返回的是 `slice`，`assembly RETURN(0, uint256.max)` 会怎样?

``` go
func opReturn(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
	offset, size := scope.Stack.pop(), scope.Stack.pop()
	ret := scope.Memory.GetPtr(int64(offset.Uint64()), int64(size.Uint64()))

	return ret, nil
}

// GetPtr returns the offset + size
func (m *Memory) GetPtr(offset, size int64) []byte {
	if size == 0 {
		return nil
	}

	if len(m.store) > int(offset) {
		return m.store[offset : offset+size]
	}

	return nil
}
```

解答

在执行 `operation.execute` 前，会根据 `operation.memorySize(stack)` 得到内存大小，然后 `mem.Resize(memorySize)` 扩容内存，所以不会越界

而且，在内存扩容前，会根据 `operation.dynamicGas()` 先扣除 gas，所以不会无端扩用内存

``` go
// go-ethereum/core/vm/interpreter.go
func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
	for {
		op = contract.GetOp(pc)
		operation := in.cfg.JumpTable[op]

		var memorySize uint64
		// calculate the new memory size and expand the memory to fit
		// the operation
		// Memory check needs to be done prior to evaluating the dynamic gas portion,
		// to detect calculation overflows
		if operation.memorySize != nil {
			memSize, overflow := operation.memorySize(stack)
			if overflow {
				return nil, ErrGasUintOverflow
			}
			// memory is expanded in words of 32 bytes. Gas
			// is also calculated in words.
			if memorySize, overflow = math.SafeMul(toWordSize(memSize), 32); overflow {
				return nil, ErrGasUintOverflow
			}
		}

		// Dynamic portion of gas
		// consume the gas and return an error if not enough gas is available.
		// cost is explicitly set so that the capture state defer method can get the proper cost
		if operation.dynamicGas != nil {
			var dynamicCost uint64
			dynamicCost, err = operation.dynamicGas(in.evm, contract, stack, mem, memorySize)
			cost += dynamicCost // total cost, for debug tracing
			if err != nil || !contract.UseGas(dynamicCost) {
				return nil, ErrOutOfGas
			}
		}

		if memorySize > 0 {
			mem.Resize(memorySize)
		}

		// execute the operation
		res, err = operation.execute(&pc, in, callContext)
	}
	return nil, nil
}
```

## API 入口

### 注册

获取 API

``` zsh
github.com/ethereum/go-ethereum/internal/ethapi.NewPublicBlockChainAPI at api.go:606
github.com/ethereum/go-ethereum/internal/ethapi.GetAPIs at backend.go:106
github.com/ethereum/go-ethereum/eth.(*Ethereum).APIs at backend.go:295
github.com/ethereum/go-ethereum/eth.New at backend.go:256
github.com/ethereum/go-ethereum/cmd/utils.RegisterEthService at flags.go:1711
main.makeFullNode at config.go:162
main.localConsole at consolecmd.go:79
github.com/ethereum/go-ethereum/cmd/utils.MigrateFlags.func1 at flags.go:1953
gopkg.in/urfave/cli%2ev1.HandleAction at app.go:490
gopkg.in/urfave/cli%2ev1.Command.Run at command.go:210
gopkg.in/urfave/cli%2ev1.(*App).Run at app.go:255
main.main at main.go:260
runtime.main at proc.go:225
runtime.goexit at asm_amd64.s:1371
 - Async Stack Trace
runtime.rt0_go at asm_amd64.s:226
```

``` go
// go-ethereum/internal/ethapi/backend.go
func GetAPIs(apiBackend Backend) []rpc.API {
	nonceLock := new(AddrLocker)
	return []rpc.API{
		{
			Namespace: "eth",
			Version:   "1.0",
			Service:   NewPublicEthereumAPI(apiBackend),
			Public:    true,
		}, {
			Namespace: "eth",
			Version:   "1.0",
			Service:   NewPublicBlockChainAPI(apiBackend),
			Public:    true,
		}, {
			Namespace: "eth",
			Version:   "1.0",
			Service:   NewPublicTransactionPoolAPI(apiBackend, nonceLock),
			Public:    true,
		}, {
			Namespace: "txpool",
			Version:   "1.0",
			Service:   NewPublicTxPoolAPI(apiBackend),
			Public:    true,
		}, {
			Namespace: "debug",
			Version:   "1.0",
			Service:   NewPublicDebugAPI(apiBackend),
			Public:    true,
		}, {
			Namespace: "debug",
			Version:   "1.0",
			Service:   NewPrivateDebugAPI(apiBackend),
		}, {
			Namespace: "eth",
			Version:   "1.0",
			Service:   NewPublicAccountAPI(apiBackend.AccountManager()),
			Public:    true,
		}, {
			Namespace: "personal",
			Version:   "1.0",
			Service:   NewPrivateAccountAPI(apiBackend, nonceLock),
			Public:    false,
		},
	}
}
```

启动 RPC 服务

``` zsh
github.com/ethereum/go-ethereum/node.RegisterApis at rpcstack.go:520
github.com/ethereum/go-ethereum/node.(*httpServer).enableRPC at rpcstack.go:283
github.com/ethereum/go-ethereum/node.(*Node).startRPC at node.go:364
github.com/ethereum/go-ethereum/node.(*Node).openEndpoints at node.go:269
github.com/ethereum/go-ethereum/node.(*Node).Start at node.go:173
github.com/ethereum/go-ethereum/cmd/utils.StartNode at cmd.go:70
main.startNode at main.go:332
main.localConsole at consolecmd.go:80
github.com/ethereum/go-ethereum/cmd/utils.MigrateFlags.func1 at flags.go:1953
gopkg.in/urfave/cli%2ev1.HandleAction at app.go:490
gopkg.in/urfave/cli%2ev1.Command.Run at command.go:210
gopkg.in/urfave/cli%2ev1.(*App).Run at app.go:255
main.main at main.go:260
runtime.main at proc.go:225
runtime.goexit at asm_amd64.s:1371
 - Async Stack Trace
runtime.rt0_go at asm_amd64.s:226
```

``` go
// go-ethereum/node/rpcstack.go
func RegisterApis(apis []rpc.API, modules []string, srv *rpc.Server, exposeAll bool) error {
	if bad, available := checkModuleAvailability(modules, apis); len(bad) > 0 {
		log.Error("Unavailable modules in HTTP API list", "unavailable", bad, "available", available)
	}
	// Generate the allow list based on the allowed modules
	allowList := make(map[string]bool)
	for _, module := range modules {
		allowList[module] = true
	}
	// Register all the APIs exposed by the services
	for _, api := range apis {
		if exposeAll || allowList[api.Namespace] || (len(allowList) == 0 && api.Public) {
			if err := srv.RegisterName(api.Namespace, api.Service); err != nil {
				return err
			}
		}
	}
	return nil
}
```

### 回调

``` zsh
curl 'http://127.0.0.1:8545/' \
  -H 'sec-ch-ua: "Google Chrome";v="93", " Not;A Brand";v="99", "Chromium";v="93"' \
  -H 'Referer: ' \
  -H 'DNT: 1' \
  -H 'sec-ch-ua-mobile: ?0' \
  -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/93.0.4577.63 Safari/537.36' \
  -H 'sec-ch-ua-platform: "macOS"' \
  -H 'Content-Type: application/json' \
  --data-raw '{"jsonrpc":"2.0","id":20099,"method":"eth_estimateGas","params":[{"from":"0xf8803a46e6ed2fb1d5ae3fddec7d1ba73a8df3e4","to":"0x90e960d70318a6dda076907486efa55656ccb2e7","data":"0x43dd776c","value":"0x0"}]}' \
  --compressed
```

``` zsh
github.com/ethereum/go-ethereum/internal/ethapi.(*PublicBlockChainAPI).EstimateGas at api.go:1113
reflect.Value.call at value.go:476
reflect.Value.Call at value.go:337
github.com/ethereum/go-ethereum/rpc.(*callback).call at service.go:206
github.com/ethereum/go-ethereum/rpc.(*handler).runMethod at handler.go:389
github.com/ethereum/go-ethereum/rpc.(*handler).handleCall at handler.go:337
github.com/ethereum/go-ethereum/rpc.(*handler).handleCallMsg at handler.go:298
github.com/ethereum/go-ethereum/rpc.(*handler).handleMsg.func1 at handler.go:139
github.com/ethereum/go-ethereum/rpc.(*handler).startCallProc.func1 at handler.go:226
runtime.goexit at asm_amd64.s:1371
 - Async Stack Trace
github.com/ethereum/go-ethereum/rpc.(*handler).startCallProc at handler.go:222
```

``` go
// go-ethereum/rpc/service.go
func (r *serviceRegistry) callback(method string) *callback {
	elem := strings.SplitN(method, serviceMethodSeparator, 2)
	if len(elem) != 2 {
		return nil
	}
	r.mu.Lock()
	defer r.mu.Unlock()
	return r.services[elem[0]].callbacks[elem[1]]
}

// go-ethereum/internal/ethapi/api.go
func (s *PublicBlockChainAPI) EstimateGas(ctx context.Context, args TransactionArgs, blockNrOrHash *rpc.BlockNumberOrHash) (hexutil.Uint64, error) {
	bNrOrHash := rpc.BlockNumberOrHashWithNumber(rpc.PendingBlockNumber)
	if blockNrOrHash != nil {
		bNrOrHash = *blockNrOrHash
	}
	return DoEstimateGas(ctx, s.b, args, bNrOrHash, s.b.RPCGasCap())
}
```

