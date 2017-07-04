# react优化 [原文](https://auth0.com/blog/optimizing-react/?utm_source=mybridge&utm_medium=blog&utm_campaign=read_more)

react的核心团队花费大量的时间和金钱来确保React只对发生变化的DOM进行更改。然而，作为开发人员，我们需要注意，我们编写的代码和我们写的方式对我们的应用程序的性能有很大的影响。我们不能期望框架帮我们解决一切问题。

在进行更新时，React中的reconciliation 过程可能相当复杂。推荐你阅读相关的文章[ Facebook's article on reconciliation in React](https://facebook.github.io/react/docs/reconciliation.html)

在react中可能会出现两种导致浪费的操作。
* 一是计算不会改变的虚拟DOM的片段。
* 二是在不需要更改的时候更改实际的DOM。

我们现在尝试对一个小型应用进行优化。在这个应用中可以通过输入值来改变方块的背景颜色，在输入框的下方有多个盒子，里面写着着星战角色的名字。我们首先将一些数据打印到控制台，以便我们可以判断我们的优化是否正常，之后我们将实现不同的优化方法。

## 开始
首先，我们需要设置一些东西，你可以[从github克隆仓库下来](https://github.com/searsaw/optimizing-react)。
启动应用程序，如图的显示
![](/img/initial_view.png) 

## 分析
现在我们已经在电脑上安装了应用，我们需要一个方式去分析react运行所消耗的时间。幸运的是，Facebook上的团队已经创建了一个名为react-addons-perf的包。 使用起来很简单 当我们准备开始分析时，我们调用Perf.start（）。 我们执行一些操作来配置文件，然后调用Perf.stop（）。 一旦我们有一些分析数据，我们可以输出几个表来查看这些数据。 我们将集中精力的是Perf.printWasted（）和Perf.printOperations（）。在这种情况下，多余的计算意味着，如前所述，React计算出新的虚拟DOM，将其与旧的DOM进行比较，没有发生变化的部分也被纳入计算的范围。这意味着它浪费了创建新的虚拟DOM的时间，因为页面中的一些部分根本不会改变。Operations是对实际的DOM做出的反应，使其反映虚拟DOM。

我已经在`src/components/PerfProfiler/index.js`目录下创建一个`react-addons-perf`的组件，以便于方便调用。

```
import React from 'react';
import Perf from 'react-addons-perf';
import styles from './styles.css';

class PerfProfiler extends React.Component {
  constructor(props) {
    super(props);

    this.state = { started: false };
  }

  toggle = () => {
    const { started } = this.state;

    started ? Perf.stop() : Perf.start();

    this.setState({ started: !started });
  }

  printWasted = () => {
    const lastMeasurements = Perf.getLastMeasurements();

    Perf.printWasted(lastMeasurements);
  }

  printOperations = () => {
    const lastMeasurements = Perf.getLastMeasurements();

    Perf.printOperations(lastMeasurements);
  }

  render() {
    const { started } = this.state;

    return <div className={styles.perfProfiler}>
      <h1>Performance Profiler</h1>
      <button onClick={this.toggle}>{started ? 'Stop' : 'Start'}</button>
      <button onClick={this.printWasted}>Print Wasted</button>
      <button onClick={this.printOperations}>Print Operations</button>
    </div>;
  }
}

export default PerfProfiler;
```

style目录 `src/components/PerfProfiler/styles.css` 

```
.perf-profiler {
  display: flex;
  flex-direction: column;
  position: absolute;
  right: 50px;
  top: 20px;
  padding: 10px;
  background: #bada55;
  border: 2px solid black;
  text-align: center;
}

.perf-profiler > h1 {
  font-size: 1.5em;
}

.perf-profiler > button {
  display: block;
  margin-top: 10px;
  padding: 5px;
}
```

然后在`src/components/App.js`引入组件
```
import PerfProfiler from './PerfProfiler'; // This import should be at the top with the rest

...

return <div id="container">
  <PerfProfiler /> <!-- Add this line to put the profiler on the page --->
  <div id="form-container">
    <Form onSubmit={this.onFormSubmit} />
```

如果你重新启动服务，你会看见在应用中多了一个操作界面，我们会通过打开和关闭来分析，并在控制台中打印出数据。
![](/img/perf_in_action.png) 

如果你按下'start'，改变颜色，点击'stop',然后点击'print Wasted',你会看见控制台中打印所有多余的虚拟dom计算。

## 在列表中 key的重要性

如果你打开控制台，你会看见来自react的警告，告诉你在列表的每个item中需要一个`key`属性，这样react才能识别单独的item。
在`src/components/CharacterList/index.js`中加上key
```
{characters.map((c, i) =>
  <Character
    key={i}
    character={c}
    style={getStyles(color, i)}
    onClick={this.removeCharacter(i)} />
)}
```

这样就不会发出警报了。现在让我们看看，当点击item的时候，有多少多余的计算操作。为了分析这个过程，我们先要点击'start',然后点击item。你最好点击一个最上面的item,然后点击'stop'。
点击'print Wasted'看看刚才的操作有多少无用的计算操作。instance 和render counts 相当于 删除之前的字符数。 我们稍后会解决这个多余计算。

现在点击'print operations'，看看出现了多少次操作
![](/img/bad_key_wasted_operations.png) 

我的天，当删除一个元素的时候一共进行了84次操作。这好像不太正常。我们再仔细看一下啊控制台的信息，可以看到很多替换文本操作，但是我们并没有让react更新任何文本。
但事实就是，我们无意间进行了文本替换操作。React 通过 `key`属性来标识在列表中的item。React的工作原理是渲染虚拟DOM表示的。这本质上是一个javascript对象，当一些东西发生改变的时候，他会重新渲染，并且把新的对象和旧的对象进行比较。无论真实dom发生了什么变化。如果我们删除时使用数组的索引作为关键字迭代的数组中的一个项目，那么下一个渲染DOM的索引将会通通减一。
```
['one', 'two', 'three', 'four', 'five']
          ^
          |-- let's remove this one

['one', 'three', 'four', 'five']
```
第一个数组中的`one`的索引是0,`two`的索引是1，`three`的索引为2，以此类推。当我们移除`two`时，在`two`后面的所有元素的索引将会向前移动一位。当react通过`key`进行对比的时候，他会发现索引对应的文本发生了变化，所以在删除的项目之后的所有项目就会进行“文本替换操作”。那么它会删除最后一个，因为它认为这个是被删除的。
我们需要通过一些方式来让react只移除被点击的那个项目，并且影响列表中的其他项目。我们可以通过添加一个唯一key来避免重新渲染。可以使用后台同步的数据来作为key

```
...
<Character
  key={c.name}
  character={c}
  style={getStyles(color, i)}
...
```

我们只是简单地改变了键，而不是等于c.name，这将是我们正在迭代的字符的名称。描述删除一个字符，然后看看操作。你将看到我们已经成功地修复了所有的“替换文本”操作，但是又出现另外一些新的替换操作。

## 实现 `shouldComponentUpdate`
现在又出现了大量不应该出现的"update styles"的操作。我们想要默认颜色始终为白色。然而，自从我们返回一个空的字符串，react会把一个空的字符串转换成'white',并且自动转换成`background-color`。我们不希望是这样的操作，可以返回一个指定颜色，省去转换操作从而带来多的余计算。

```
const getStyles = (color, index) => {
  if (index % 3 === 0) {
    return { backgroundColor: color };
  }

  return { backgroundColor: 'white' };
};
```

现在使用PerfProfiler配置删除列表项并输出操作。 所有这些“update styles”的操作都没有了。 现在通过在PerfProfiler上按“Print Wasted”来打印多余的操作。
![](/img/update_styles_fixed.png) 

还是出现了多余虚拟DOM计算，我们需要告诉react，当`Character`接受新的props才会重新渲染。

react 组件默认是一直重新渲染的。然而，我们可以通过`shouldComponentUpdate`来告诉react是否要重新渲染。这是我们在传递新props到组件中和在应用程序中发生更改时创建的新状态的一种方法。果我们返回true，那么它将重新渲染该函数;值为false会阻止它。我们会用这个来比较组件当前的props的接收新的props.

```
shouldComponentUpdate(nextProps) {
  const { character, style, onClick } = this.props;

  return character.name !== nextProps.character.name
    || style.backgroundColor !== nextProps.style.backgroundColor;
}
```
上面意思是说，如果字符名称或样式颜色不匹配，我们要重新渲染组件。在浏览器中配置文件。 你会看到我们没有出现多余的计算操作！ 但是，该解决方案不能很好地扩展。 如果我们添加另一个参数或者更改onClick，那么我们需要确保将检查添加到我们的`shouldComponentUpdate`方法。 React给我们一个自动的方法。 此逻辑存在于React.PureComponent内, 我们可以从该组件扩展我们的组件.

## 使用 `React.PureComponent`

这个方法只提供浅对比。对于简单的类型，如数字和字符串，这不是问题，但当我们传递对象或数组时，这成为一个问题。对象或数组中的数据更改将不会被自动提取，因为被比较的prop为对象或数组的引用。浅的比较没有对比到它内部的数据。

```
const obj1 = { name: "George" };
const obj2 = { name: "George" };
const obj3 = obj1;

obj1 === obj2 // false
obj2 === obj3 // false
obj1 === obj3 // true
```

另请注意，如果其父级以某种方式显示更改，则React将仅检查组件是否有更改。 这意味着如果一个组件从shouldComponentUpdate返回false，或者在扩展PureComponent时不显示支持更改，那么它们的子项也不会被检查。 因此，重要的是，在使用PureComponent时，所有的子项也需要是纯组件

`Character`实现

```
import React from 'react';
import styles from './styles.css';

class Character extends React.PureComponent {
  onClick = () => {
    const { character, onClick } = this.props;

    onClick(character.name);
  }

  render() {
    const { character, backgroundColor } = this.props;
    const style = { backgroundColor };

    return <div
      className={styles.character}
      style={style}
      onClick={this.onClick}>
      <p>{character.name}</p>
    </div>;
  }
}

export default Character;
```

`CharacterList`实现
```
import React from 'react';
import Character from '../Character';
import styles from './styles.css';

class CharacterList extends React.Component {
  constructor(props) {
    super(props);

    this.state = {
      characters: this.props.characters.slice(0)
    };
  }

  removeCharacter = characterName => {
    const { characters } = this.state;
    const characterIndex = characters.findIndex(c => c.name === characterName);

    characters.splice(characterIndex, 1);

    this.setState({ characters });
  }

  render() {
    const { characters } = this.state;
    const { color } = this.props;

    return <div className={styles.characterList}>
      {characters.map((c, i) =>
        <Character
          key={c.name}
          character={c}
          backgroundColor={i % 3 === 0 ? color : 'white'}
          onClick={this.removeCharacter} />
      )}
    </div>
  }
}

export default CharacterList;
```

我们已经删除了getStyles函数，因为它不再需要。 另外，我们更新了removeCharacter是一个简单的方法，它取名字，从字符列表中删除该名称的字符，然后用该列表更新状态。 最后，我们更新我们的渲染功能，以便传递正确的props。 注意我已经使用一个三元运算符来获取backgroundColor的必需值。 这可以被抽象到自己的功能中，但是我觉得这是小而容易理解的，不能保证。
现在描述一个字符的删除。我们没有更多的浪费操作！在我看来，我也认为这个代码对于一个新的代码库来说更清洁，更容易理解。

## 使用不可变对象
我们已经做了一大堆优化，但还有待改进。 尝试将颜色更改为“红色”。 然后在输入框中再次键入“红色”。 不要点击“更改颜色”！ 在分析器上按“开始”。 接下来，点击“更改颜色”。 最后，点击“停止”。 我们将一些方格的颜色改为红色，然后再次更改