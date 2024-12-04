
任务目标：设计一个简单的投票统计器

1. 设计一个简单的投票统计器用于小团队内部投票，要求能累积统计出赞成票和反对票的票数。
2. 考虑检查投票者属于团队成员，假设队员不会重复投票。


### ZkProgram

[ZkProgram()](https://docs.minaprotocol.com/zkapps/o1js/recursion) 是Mina递归电路执行的入口，你可以使用该工具执行和验证递归零知识证明，整个证明生成过程全部都是链下计算完成。我们从示例代码角度来了解一下`ZkProgram()`

```ts
import { Field, ZkProgram } from 'o1js';

const SimpleProgram = ZkProgram({
  name: 'simple-program-example',
  publicInput: Field,

  methods: {
    run: {
      privateInputs: [],

      async method(publicInput: Field) {
        publicInput.assertEquals(Field(0));
      },
    },
  },
});

const { verificationKey } = await SimpleProgram.compile();
const { proof } = await SimpleProgram.run(Field(0));
```

可以看到`ZkProgram()`本身是一个高阶函数，它接收一个配置对象作为参数（模板DLS），并且返回值也是一个包含多个函数的对象：

```ts
return {
    compile: () => Promise<...>,
    verify: () => Promise<...>,
    // ... other methods
}
```

根据上述内容可以看到，整个电路全生命周期从初始化到证明生成和验证都涉及`ZkProgram()`。

接下来从Typescript类型约束角度看一下如何编写输入参数模板DLS。

```ts
Config extends {
    publicInput?: ProvableTypePure;    
    publicOutput?: ProvableTypePure;   
    methods: {                         
        [I in string]: {
            privateInputs: Tuple<PrivateInput>;
            auxiliaryOutput?: ProvableType;
        };
    };
}
```

这是基础输入约束，输入的methods对象可以定义任意的string，上述示例使用的是 `run`，但这里面必须包含`privateInputs`，还有`method`函数，关于该函数的约束如下

```ts
Methods extends {
    [I in keyof Config['methods']]: Method<
        InferProvableOrUndefined<Get<Config, 'publicInput'>>,
        InferProvableOrVoid<Get<Config, 'publicOutput'>>,
        Config['methods'][I]
    >;
}
```

`method` 函数的类型约束通过 `Method` 泛型来定义，它接收三个类型参数：

```ts
Method<
    PublicInput,   // derive from Config['publicInput']
    PublicOutput,  // derive from Config['publicOutput']
    MethodConfig   // derive from Config['methods'][I]
>
```

所以上述示例中`method`传入的第一个参数是`publicInput`。该方法的返回值类型是一个Promise，所以使用的时候需要加`async`，其他的`compile`，`verify`等方法返回值也都是Promise，所以也需要加`async`。

为什么要使用Promise？在o1js源代码的`lib/proof-system/workers.js`有这样一段内容：

> Set the number of workers to use for parallelizing the proof generation. By default the number of workers is set to the number of physical CPU cores on your machine.

零知识证明的生成是计算密集型任务，涉及复杂的密码学运算，如果在主线程执行会阻塞 UI 和其他操作。所以通常在worker线程计算，worker线程和主线程之间通信需要用到promise。

### Struct

Struct可以让你使用o1js定义复合类型。我们以"Voter" struct为例看一下如何具体定义struct。

```ts
let Vote = { hasVoted: Bool, inFavor: Bool };

class Voter extends Struct({
  publicKey: PublicKey,
  votes: [Vote, Vote, Vote]
}) {
  vote(index: number, inFavor: Bool) {
    let vote = this.votes[index];
    vote.hasVoted = Bool(true);
    vote.inFavor = inFavor;
  }
}

let emptyVote = { hasVoted: Bool(false), inFavor: Bool(false) };
let voter = new Voter({ publicKey, votes: Array(3).fill(emptyVote) });
voter.vote(1, Bool(true));
```

该数据结构表明Voter可以对3个不同的vote提支持或者反对票。 `publicKey` 是用来标识和验证投票者身份的。


## Task2

代码拷贝自[mina-sz](https://github.com/coldstar1993/mina-sz/blob/main/contracts/src/others/program-on-vote.ts)，你也可以在本仓库中找到一样的代码。

整体代码分成3块，Struct复合数据类型定义，zkProgram电路结构和递归证明部分。

`VoteState`定义该如何投票。

`assertInitialState()`方法确保`VoteState`数据初始化时正常。

`applyVote()`方法是投票的具体逻辑。

在该函数中首先检查投票状态，要求在新投票之前现有投票数要小于`MEMBER_CNT`。之后比较成员资格，要求投票成员的public key和`VoteState`初始化时数据一致。随后执行投票动作返回一个新的VoteState。

为什么要投票时采用更新VoteState方式提供证明？这个目的是为了实现递归零知识证明，每一次状态变更都提供一次证明。

在这个投票系统中：

1. 每次投票都会：
    
    - 使用上一次的证明（`earlierProof`）
    - 生成一个新的证明
    - 这个新证明包含了最新的投票状态
2. 证明链的形成：

```
初始状态证明 -> 第一次投票证明 -> 第二次投票证明 -> 第三次投票证明 ...
```

`checkEqual`是两个State相等性比较。这是什么意思，为什么要做相等性比较？

因为电路实际上是用来验证结果而不是计算然后输出。所以需要外部有一个计算最终结果输入，然后和电路的计算结果比较，生成证明从而证明外部的输入是正确可信的。

o1js这方面代码写的有一些晦涩，用Halo2举例子会更直观一点。

```rust
struct MyCircuit<F: Field> {
	// Private input
	private_input: Value<F>,
	// Public input
	public_input: F,
	// Expected output
	expected_output: F,
}

impl<F: Field> Circuit<F> for MyCircuit<F> {
    fn synthesize(...) {
        // Perform computation in the circuit
	let result = /* Circuit computation logic */;

        // Verify that computation result matches expected output
	cs.enforce_equal(result.cell(), expected_output.cell());
    }
}
```

但是o1js没有这种直观写法，使用o1js大概需要写成这样的形式

```js
// Compute expected result outside the circuit
const expectedState = new VoteState({...});

// Pass the expected result as the first parameter
const proof = await VoteProgram.initVoteState(expectedState);
```

`checkEqual`相等性比较就是比较电路输入和外部输入是否相同（你可以把外部输入当成是Halo2的`expected_output`）。

电路和递归证明部分比较直观就不做解释了。
