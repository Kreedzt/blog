---
author: "Kreedzt"
title: "React hooks 入门"
date: "2023-02-02"
description: "React hooks 简明入门教程"
tags: ["web", "javascript", "react.js"]
draft: false
---

# React hooks 入门

> 此教程翻译自Academind课程, 源地址:[https://www.youtube.com/watch?v=-MlNBTSg_Ww](https://www.youtube.com/watch?v=-MlNBTSg_Ww)

## 为何需要React Hooks
为了状态和生命周期而去写Class组件往往是繁琐的, 而且组合优于继承的原则, 使得在多数情况下能定义函数组件就不去定义类组件
在新版React(16.8)中, 新增了React Hooks, 使我们可以在函数组件中使用状态和生命周期函数, 使得函数组件更加灵活, 减轻类组件使用负担

---


## 从state开始(useState)

```
项目目录:
src/
	|-components/
  |---TabList.js
  |---Schedule.js
  |-hooks/
  |---http.js
```

首先我们声明一个无状态函数组件, 内容如下:

```javascript
const TabList = () => {
  const currentTab = 1;
  return <div>
      CurrentTab: {currentTab}
    </div>;
};
```
这是一个普通的无状态组件, 我们使用新的useState API来赋予初始值: (useState的调用参数和state一样, 可赋予任何值)(useState的参数为state的初始值)
useState方法返回1个数组, 数组内包含2个参数, 第一个参数是当前state:

```javascript
import React, { useState } from 'react';

const TabList = () => {
  const [state] = useState({
    selectedTab: 1
  });
  return <div>
    CurrentTab: {state.selectedTab}
    </div>
}
```
useState方法返回1个数组, 第二个参数是改变状态函数(此处暂时理解为setState, 但≠setState)

```javascript
const TabList = () => {

  const [tabState, setTabState] = useState(1);


  const changeTabState = () => {
    setTabState(tableState + 1);
  };

  return <div>
              CurrentTab: {tabState}
              <button onClick={changeTabState}>
                Just CurrentTab + 1
              </button>
            </div>;
};
```
刷新浏览器即可看到setTableState'模仿'了class组件中的setState方法.
为什么说模仿呢 ? 我们修改下useState的参数, 现在传递一个多属性的对象值, 如下:

```javascript
const TabList = () => {

  const [tabState, setTabState] = useState({
    selectedTab: 1,
    text: 'initValue'
  });


  const changeTabState = () => {
    setTabState({
      selectedTab: tabState.selectedTab + 1
    });
  };

  return <div>
           <div>
  					CurrentTab: {tabState.selectedTab}
           </div>
           <div>
              <button onClick={changeTabState}>
                Just CurrentTab + 1
  							</button>
           </div>
           CurrentText: {tabState.text}
   		</div>;
};
```
刷新浏览器: 第一次text的state加载正常, 我们点击按钮, text消失了???

### useState引发的问题

从上段代码中, 我们可以发现useState返回的参数中, 使用修改状态的函数时候, 跟class中的setState并不一致.
在class中, 我们每次使用setState进行改变状态的时候, React做出的操作是mergeData, 我们简单理解为Object.assign方法, 而在useState返回值中, 我们修改是直接replaceData, 是直接替换了状态, 替换了新的值上去, 所以, 上述代码中`text`的值在更新状态后丢失了, 导致渲染空数据.

### useState问题的解决方案

#### 手动mergeData

就像redux那样, 我们每次返回值的时候把上一次的值再传递过去, 可以很快的解决旧数据丢失问题:

```javascript
  const changeTabState = () => {
    setTabState({
      ...tabState,
      selectedTab: tabState.selectedTab + 1
    });
  };
```

#### 定义多个state
在class中, 我们有且仅有一个state是因为state是作为class的属性而存在的, 而在函数组件中, useState是一个方法, 我们不难想到, 可以定义多个state去分别更新不同的state, 这也是React的灵活设计方式:

```javascript
const TabList = () => {
  const [tabState, setTabState] = useState(1);
  const [textState, setTextState] = useState('initial value');

  const changeTabState = () => {
    setTabState(tabState + 1);
  };

  const changeTextState = () => {
    setTextState('new value');
  };

  return (
    <React.Fragment>
      <div>CurrentTab: {tabState}</div>
      <div>
        <button onClick={changeTabState}>Just CurrentTab + 1</button>
      </div>
      <div>CurrentText: {textState}</div>
      <div>
        <button onClick={changeTextState}>set new Text Value</button>
      </div>
    </React.Fragment>
  );
};
```

---


## 生命周期函数useEffect
在React hooks中, 所有生命周期函数都将由useEffect这一个API实现

### componentDidMount
我们通常在componentDidMount函数中发起网络请求, 确认接收到参数再渲染数据.
我们使用useEffect函数来实现此场景:
useEffect是一个函数, 可接收一个方法
我们定义一个新的组件作为TabList的子组件, 来测试是否还原了'componentDidMount'方法:
```javascript
const TabList = () => {
  const [clickTimeState, setClickTimeState] = useState(0);

  const changeClickTime = () => {
    setClickTimeState(clickTimeState + 1);
  };

  return (
    <React.Fragment>
      <div>ClickTime: {clickTimeState}</div>
      <div>
        <button onClick={changeClickTime}>Just clickTime + 1</button>
      </div>
      <Schedule clickTime={clickTimeState} />
    </React.Fragment>
  );
};
```

```javascript
import React, { useEffect } from 'react';

const Schedule = ({ clickTime }) => {
  useEffect(() => {
    console.log('clickTime', clickTime);
    console.log('useEffect works');
  });

  return <React.Fragment>This is Schedule</React.Fragment>;
};
```
我们多次点击按钮, 发现每次都执行了useEffect中的代码, 现在, 证明了一点: useEffect中的函数在每次props改变都会执行.
在componentDidMount中, 自己状态改变只会执行一次, 如果组件内状态改变了, 但是useEffect中的函数代码没有执行, 就证明我们'还原'了componentDidMount这个API.
我们现在模拟一个网络请求来修改组件内状态:

```javascript
const actions = ['eat', 'sleep'];

let pointer = 0;

const newScheduleList = [];
for (let i = 0; i < 10; i++) {
  if (actions[pointer]) {
    newScheduleList.push({
      id: i,
      action: actions[pointer]
    })
  } else {
    newScheduleList.push({
      id: i,
      action: actions[0]
    });
    pointer = 0;
  }
  pointer++; 
}

// clickTime后续会用到
const Schedule = ({clickTime}) => {
  const [loading, setLoading] = useState(false);
  const [scheduleList, setScheduleList] = useState([]);

  useEffect(() => {
    setLoading(true);
    console.log('useEffect works');
    setTimeout(() => {
      setLoading(false);
      setScheduleList(newScheduleList);
    }, 1000);
  });

	return (
    <React.Fragment>
      {loading ? (
        'loading...'
      ) : (
        <ul>
          {scheduleList.map((item) => (
            <li key={item.id}>
              {item.id} -- {item.action}
            </li>
          ))}
        </ul>
      )}
    </React.Fragment>
  );
};
```
    刷新浏览器, 打开控制台, 我们发现进入了一个死循环, useEffect内的函数在无限制执行
>     分析后不难发现, useEffect好像在每次componentDidUpdate执行, props不变的情况下, 我们修改了setLoading, 此时state变化, 第二次进入useEffect内函数代码, 所以陷入了一个死循环

useEffect接受2个参数, 第一个为函数, 第二个为数组, 第二个参数用来监听dependencies, 每次数组内的值变
化, 触发第一个函数, 默认监听所有dependencies, 第一次render必定执行useEffect函数.
现在, 我们第二个参数传递一个空数组: [], 即为不监听任何变化, 每次state/props改变不重新执行第一个参数的函数, 刷新浏览器, useEffect内函数的代码只执行了一次

```javascript
  useEffect(() => {
    setLoading(true);
    console.log('componentDidMount ?');
    setTimeout(() => {
      setLoading(false);
      setScheduleList(newScheduleList);
    }, 1000);
  }, []);
```
我们至此还原了componentDidMount

---


### componentDidUpdate

通常在class组件中, 我们对props传递过来的id不同而做一次网络请求,,如下:

```javascript
componentDidUpdate(prevProps) {
  if (prevProps.selectedId !== props.selectedId) {
    // ... send xhr
    this.fetchData();
  }
}
```
为了实现这个业务, 我们仅需在useEffect方法第二个参数传递指定的监听值即可, 不需要再次调用一个useEffect, 否则会在第一次渲染时候触发2次fetchData

```javascript
  const fetchData = () => {    
    setLoading(true);
    console.log('triggered fetchData');
    setTimeout(() => {  
      setLoading(false);
      setScheduleList(newScheduleList);
    }, 1000);
  };
  
  useEffect(() => {
    console.log('fetchData when clickTime diff');
    fetchData();
  }, [clickTime])
```
这样, 每次点击按钮就触发了重新请求

---


### componentWillUnmount
通常我们在componentWillUnmount里移除事件监听, 取消Obvervable订阅, 以便及时清理内存
useEffect接受函数可以return一个函数, 作为componentWillUnmount的实现:
Schedule组件修改代码:

```javascript
  useEffect(() => {
    console.log('fetchData when clickTime diff');
    fetchData();
    return () => {
      console.log('componentWillUnmount ?');
    }
  }, [clickTime])
```
TabList组件代码:

```javascript
const TabList = () => {
  const [clickTimeState, setClickTimeState] = useState(0);

  const [showSchedule, setShowSchedule] = useState(true);

  const changeClickTime = () => {
    setClickTimeState(clickTimeState + 1);
  };

  const changeShowSchedule = () => {
    setShowSchedule(false);
  }

  return (
    <React.Fragment>
      <div>ClickTime: {clickTimeState}</div>
      <div>
        <button onClick={changeClickTime}>Just clickTime + 1</button>
      </div>
      <div>
        <button onClick={changeShowSchedule}>
          Unmount Schedule
        </button>
      </div>
      {showSchedule ? <Schedule clickTime={clickTimeState} /> : ''}
    </React.Fragment>
  );
};
```
点击Unmout Schedule按钮, 控制台输出了Schedule组件的console代码
看起来好像是完美'还原', 但是点击clickTime按钮, 发现了新的问题: 控制台依旧输出了`componentWillUnmount?`的代码
> useEffect里定义的返回函数看似在每次unmount时候执行, 实际上当状态改变时, 依旧会执行此返回函数, 因为我们监听了clickTime的变化, 这样操作相当浪费, 不符合预期效果

为了避免浪费的内存, 我们再次调用一次useEffect方法, 第二个参数传递一个空数组, 表示不监听任何值变化, 此时, Schedule组件内有2个useEffect方法调用:

```javascript
  useEffect(() => {
    console.log('fetchData when clickTime diff');
    fetchData();
    // return () => {
    //  console.log('componentWillUnmount ?');
    // }
  }, [clickTime]);

  useEffect(() => {
    return () => {
      console.log('componentWillUnmount ?');
    }
  }, []);
```
此时, 刷新浏览器, 我们多次点击clickTime按钮, componentWillUnmount的log没有执行, 点击Unmount Schedule按钮, 代码才执行, 符合预期效果

---


### shouldComponentUpdate
在class组件中, 我们通常在状态变更时候定义此函数来进行判断新旧状态, 避免不必要的重新渲染:

```javascript
shouldComponentUpdate(nextProps, nextState) {
	return nextProps.selectedId !== props.selectedId || nextState.isLoading !== this.state.isLoading;
}
```
此处就不在useEffect的可控功能内了, 在React16.6中, 有一个memo方法, 该方法可记录储存当前组件, 仅当需要的props更新才重新渲染, 我们还可在调用memo的时候使用第二个参数, 传递一个函数, 函数的参数为prevProps和nextProps.
为了实现效果, 我们在TabList组件传递一个无用的参数uselessProp给Schedule组件:
TabList组件代码:

```javascript
const TabList = () => {
  const [clickTimeState, setClickTimeState] = useState(0);

  const [showSchedule, setShowSchedule] = useState(true);

  const [color, setColor] = useState('#1890ff');

  // 定义一个无用的prop传递给Schedule组件
  const [uselessProp, setUselessProp] = useState(0);

  const changeClickTime = () => {
    setClickTimeState(clickTimeState + 1);
  };

  const changeShowSchedule = () => {
    setShowSchedule(false);
  }

  const changeColor = () => {
    setColor('cyan');
  }

  const changeUselessProp = () => {
    setUselessProp(uselessProp + 1);
  }

  return (
    <React.Fragment>
      <div>ClickTime: {clickTimeState}</div>
      <div>
        <button onClick={changeClickTime}>Just clickTime + 1</button>
      </div>
      <div>
        <button onClick={changeShowSchedule}>
          Unmount Schedule
        </button>
        <button onClick={changeColor}>
          set Color to cyan
        </button>
        <button onClick={changeUselessProp}>
          set uselessProp change
        </button>
      </div>
      {showSchedule ? <Schedule clickTime={clickTimeState} color={color} uselessProp={uselessProp} /> : ''}
    </React.Fragment>
  );
};
```
Schedule组件:

```javascript
const Schedule = ({ clickTime, color }) => {
  const [loading, setLoading] = useState(false);
  const [scheduleList, setScheduleList] = useState([]);

  const fetchData = () => {
    setLoading(true);
    console.log('triggered fetchData');
    setTimeout(() => {
      setLoading(false);
      setScheduleList(newScheduleList);
    }, 1000);
  };

  useEffect(() => {
    console.log('fetchData when clickTime diff');
    fetchData();
  }, [clickTime]);

  useEffect(() => {
    return () => {
      console.log('componentWillUnmount ?');
    };
  }, []);

  // 查看渲染次数
  // 不使用memo, 每次改变uselessProp都会重新渲染
  console.log('rendering schedule...');

  return (
    <React.Fragment>
      {loading ? (
        'loading...'
      ) : (
        <ul>
          {scheduleList.map((item) => (
            <li style={{ color: color }} key={item.id}>
              {item.id} -- {item.action}
            </li>
          ))}
        </ul>
      )}
    </React.Fragment>
  );
};

export default Schedule;
```
使用memo包装Schedule组件, 仅需修改导出代码(注意: 函数返回值与shouleComponentUpdate相反)

```javascript
// export default Schedule;
export default React.memo(Schedule, (prevProps, nextProps) => {
  // 注意: 此处的函数返回值和shouldComponentUpdate截然相反, 返回true是不重新渲染, 返回false是重新渲染
  return (
    prevProps.clickTime === nextProps.clickTime &&
    prevProps.color === nextProps.color
  );
});
```

---


# React hooks的可共享逻辑

## 自定义hooks
定义一个文件在src/hooks下,命名为http.js, 复制Schedule组件的fetchData代码:

```javascript
mport { useState } from 'react';

const fetchData = (url, data) => {
  return new Promise((res, rej) => {
    setTimeout(() => {
      if (url !== '/') {
        res({
          success: true,
          data: data
        });
      } else {
        rej({
          success: false,
          message: 'not valid url'
        });
      }
    }, 1000);
  });
};

export const useHttp = (url, data) => {
  const [loading, setLoading] = useState(false);
  const [fetchedData, setFetchedData] = useState(null);

  console.log('triggered fetchData');
  setLoading(true);
  fetchData(url, data)
    .then((res) => {
      setFetchedData(res);
    })
    .catch((err) => {
      setFetchedData(err);
    })
    .finally(() => {
      setLoading(false);
    });
  return [loading, fetchedData];
};
```
但是这个hooks是不合理的, 往往组件内逻辑是很复杂的, 我们可能要执行一次或监听不同的值而执行多次useHttp, 这样就要写不止一次的useHttp()
我们应该在useHttp方法内引用useEffect使得该自定义钩子更加灵活, 可以不必在useEffect内调用useHttp

```javascript
import { useState, useEffect } from 'react';

// mock Data
const actions = ['eat', 'sleep'];

let pointer = 0;

const newScheduleList = [];
for (let i = 0; i < 10; i++) {
  if (actions[pointer]) {
    newScheduleList.push({
      id: i,
      action: actions[pointer]
    });
  } else {
    newScheduleList.push({
      id: i,
      action: actions[0]
    });
    pointer = 0;
  }
  pointer++;
}
// end mock Data

const fetchData = (url, data) => {
  console.log('fetchData params:', url, data);
  return new Promise((res, rej) => {
    setTimeout(() => {
      if (url === '/scheduleList') {
        res({
          success: true,
          data: newScheduleList
        });
      } else {
        rej({
          success: false,
          message: 'not valid url'
        });
      }
    }, 1000);
  });
};

export const useHttp = (url, data, dependencies) => {
  const [loading, setLoading] = useState(false);
  const [fetchedData, setFetchedData] = useState(null);

  console.log('triggered fetchData');
  useEffect(() => {
    setLoading(true);
    fetchData(url, data).then(res => {
      setFetchedData(res);
    }).catch(err => {
      setFetchedData(err);
    }).finally(() => {
      setLoading(false);
    })
  }, dependencies);

  return [loading, fetchedData];
};
```
修改Schedule组件的逻辑
```javascript
import React, { useEffect } from 'react';
import { useHttp } from '../hooks/http';

const Schedule = ({ clickTime, color }) => {
  // 不必再去定义loading状态, loading现在由useHttp控制
  // const [loading, setLoading] = useState(false);
  // const [scheduleList, setScheduleList] = useState([]);

  // clickTime为我们监听的值
  const [loading, fetchedData] = useHttp('/scheduleList', { test: 1 }, [clickTime]);
  // 我们定义的fetchedData初始值为null, 一定要做判断
  const scheduleList = fetchedData ? fetchedData.data.map(item => ({
    ...item,
    success: true
  })) : [];

  // useEffect(() => {
  //   console.log('fetchData when clickTime diff');
  //  
  // }, [clickTime]);

  useEffect(() => {
    return () => {
      console.log('componentWillUnmount ?');
    };
  }, []);

  console.log('rendering schedule...');

  return (
    <React.Fragment>
      {loading ? (
        'loading...'
      ) : (
        <ul>
          {scheduleList.map((item) => (
            <li style={{ color: color }} key={item.id}>
              {item.id} -- {item.action}
            </li>
          ))}
        </ul>
      )}
    </React.Fragment>
  );
};

export default React.memo(Schedule, (prevProps, nextProps) => {
  return (
    prevProps.clickTime === nextProps.clickTime &&
    prevProps.color === nextProps.color
  );
});
```
刷新浏览器, 每次clickTime只会触发一次http请求, 完美抽离了http hooks

---


# 总结
useState: 

```javascript
const [state, setState] = useState({});
// 替换 state + setState
```

useEffect :

```javascript
// 替换componentDidMount
useEffect(() => {
  // code here...
}, [])

// 替换componentWillUnmount
useEffect(() => {
  return () => {
    // code here...
  }
}, [])

// 部分替换componentDidUpdate处理逻辑
useEffect(() => {
  // 当value1, value2变化, 执行下列代码
  // code here....
}, [value1, value2])

// 替换componentDidUpdate(不建议以下写法, 每次更新都会触发)
useEffect(() => {
  // code here...
})
```
 memo:

```javascript
// 替换shouldComponentUpdate
React.memo(ReactComponent) // 自行判断是否更新
React.memo(ReactComponent, (prevProps, nextProps) => {
  // 当id相同, 不重新渲染, 此处返回值与shouldComponentUpdate相反
  return prevProps.id === nextProps.id;
})
```
> 注意: 使用useEffect时不能嵌套在代码块中使用, React仅会在组件内寻径一层, 譬如, 以下代码不会生效:

```javascript
if (id === 0) {
  useEffect(() => {}, []) // wrong way!
}

```

