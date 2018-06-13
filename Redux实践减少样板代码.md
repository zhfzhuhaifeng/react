# Redux实践——减少样板代码

在编写Redux代码时，可能最令人感到厌烦的是必须写一大堆样板代码。在编写同步action时还能忍受，但在编写异步action时，大量的样板代码不仅使代码量陡增，而且难以维护。接下来我们就来探究如何使用`高阶函数（High-order Function）`和`中间件（middleware）`减少样板代码。

##同步action

以一个todo应用为例，具有新增、编辑、删除功能。完整代码如下：

`actions/index.js`

```javascript
export const ADD_TODO = 'ADD_TODO';
export const EDIT_TODO = 'EDIT_TODO';
export const DELETE_TODO = 'DELETE_TODO';

export const addTodo = (text) => ({
  type: ADD_TODO,
  text,
});
export const EditTodo = (id, text) => ({
  type: EDIT_TODO,
  text,
});
export const DeleteTodo = (id) => ({
  type: DELETE_TODO,
  id,
});
```

这样大量的样板代码是否令人厌烦？我们来写一个高阶函数，来生成这些action：

```javascript
function createAction(type, ...argNames) {
  return function(...args) {
    let action = { type };
    argNames.forEach((argName, index) => {
      action[argName] = args[index];
    });
    return action;
  };
}

export const ADD_TODO = 'ADD_TODO';
export const EDIT_TODO = 'EDIT_TODO';
export const DELETE_TODO = 'DELETE_TODO';

export const addTodo = createAction(ADD_TODO, 'text');
export const editTodo = createAction(EDIT_TODO, 'id', 'text');
export const deleteToto = createAction(DELETE_TODO, 'id');

// 在组件中使用
// components/App.js
dispatch(addTodo('someText'));
```

经过`createAction`这个高阶函数封装之后，编写同步action只需要一行，清晰明了。

因为action type在action和reducer中都需要引用，所以可以新建一个`actionTypes.js`，将type常量放置其中：

```
├── redux
│   ├── actions
│       ├── index.js
│   ├── reducers
│       ├── index.js
│   ├── actionTypes.js // 放置action type常量
```

`actionTypes.js`

```javascript
export const ADD_TODO = 'ADD_TODO';
export const EDIT_TODO = 'EDIT_TODO';
export const DELETE_TODO = 'DELETE_TODO';
```

在`actions/index.js`中引入：

```javascript
import * as actionTypes from '../actionTypes';

export const addTodo = createAction(actionTypes.ADD_TODO, 'text');
export const editTodo = createAction(actionTypes.EDIT_TODO, 'id', 'text');
export const deleteToto = createAction(actionTypes.DELETE_TODO, 'id');
```

再来看Reducer：

`reducers/index.js`

```javascript
import * as actionTypes from '../actionTypes';

function todos(state = [], action) {
  switch(action.type) {
    case actionTypes.ADD_TODO:
      // 只是个例子，暂时不处理id的问题
      const text = action.text.trim();
      return [...state, text];
    case actionTypes.EDIT_TODO:
      //...
    case actionType.DELETE_TODO:
      //...
    default:
      return state;
  }
}
```

满屏的`switch...case...`让人眼花，让我们来写一个生成reducer的函数：

```javascript
function createReducer(initState, handlers) {
  return function reducer(state = initState, action) {
    if (handlers.hasOwnProperty(action.type)) {
      return handlers[action.type](state, action);
    } else {
      return state;
    }
  };
}

const todos = createReducer([], {
  [actionTypes.ADD_TODO](state, action) {
    const text = action.text.trim();
    return [...state, text];
  },
  [actionTypes.EDIT_TODO](state, action) {
    //...
  },
  [actionTypes.DELETE_TODO](state, action) {
    //...
  },
});

export const rootReducer = combineReducer({
  todos,
});
```

假如`todos`中有复杂的数据处理，我们甚至能将其再次抽取出来：

```javascript
function addTodos(state, action) {
  const newTodos = state.map(todo => {
    // 复杂的数据处理
  });
  return newTodos;
}

const todos = createReducer([], {
  [actionTypes.ADD_TODO]: addTodos,
  //...
});
```

这样结构就会非常清晰，遵循`让每个函数只做一件事情`原则，当需求变更或者bug定位时，不会牵一发而动全身。

## 异步action

此处使用的异步中间件为`redux-thunk`，这是最基础的一个异步中间件。也许今后会用到`redux-saga`等更加强大的中间件，其实原理是一样的，也是使用高阶函数进行一步一步的封装。

我们来写一个最基本的异步action：

```javascript
// actionTypes.js
export const SOME_START = 'SOME_START';
export const SOME_SUCCESS = 'SOME_SUCCESS'；
export const SOME_FAILED = 'SOME_FAILED';

//actions/index.js
import * as actionTypes from '../actionTypes';

export const someAction = params => dispatch => {
  dispatch({ type: actionTypes.SOME_START });
  
  axios({
    // ...一些参数
  })
    .then(response => {
      const data = response.data.data;
      dispatch({ type: actionTypes.SOME_SUCCESS, data });
  })
    .catch(error => {
      const errorMsg = 'someText';
      dispatch({ type: actionTypes.SOME_FAILED, errorMsg });
  })
};

```

可以看到，一个最简单的异步action，需要编写的代码量就在30行左右。然而在实际项目中，我们还会有大量的业务逻辑操作，如错误提示、数据处理等等。一个界面动辄十来个几十个接口，大量的样板代码令人厌烦。那么我们怎么处理这些样板代码呢？答案是`中间件（middleware）`。

我们所使用的`redux-thunk`就是一个中间件。以下是它的源码：

```javascript
export default function thunkMiddleware({ dispatch, getState }) {
  return next => action =>
    typeof action === 'function' ?
      action(dispatch, getState) :
      next(action);
}
```

仅仅6行，就能使我们编写异步action，但这是远远不够的，接下来我们来编写自己的中间件。

我们可以看到，一个异步action涉及到3个同步action和一个AJAX请求，那么我们把它提取出来：

```javascript
export const someAction = (params) => ({
  types: [actionTypes.SOME_START, actionTypes.SOME_SUCCESS, actionTypes.SOME_FAILED],
  axiosAPI: () => axios({
    //一些参数
  }),
});
```

解释这个action的中间件可以是下面这样：

```javascript
export default ({ dispatch, getState }) => next => action => {
  const { types, axiosAPI } = action;
  
  // 不存在types属性，则交由下个中间件处理
  if (!types) {
    return next(action);
  }
  
  // 验证types是否正确的数组
  if (!Array.isArray(types) || types.length !== 3 || !types.every(type => typeof type === 'string')) {
    throw new Error('Expected types to be an array of three string types.');
  }
  
  if (typeof axiosAPI !== 'function') {
    throw new Error('Expected axiosAPI to be a function.');
  }
  
  const [startType, successType, failureType] = types;
  dispatch({ type: startType });
  
  return axiosAPI().then(response => {
    const data = response.data.data;
    dispatch({ type: successType, data });
  })
    .catch(error => {
      dispatch({ type: failureType, errorMsg: 'sometext'});
  })
}
```

当然，这个中间件并不能满足我们的实际需求。在实际项目中，一个异步请求往往伴随着消息提示，回调函数等等，如请求成功时弹出提示、关闭弹窗等等。就目前的项目而言，可以提炼出这些需求：

* 函数回调（如关闭弹窗，发起另外的请求等）
* 消息提示（包括成功、错误提示）
* 不同的errorCode有不同的提示
* 后台返回数据的处理

我们再对上面的action进行扩展：

```javascript
import { message } from 'antd'; // 引入atnd消息弹框组件

const messageCallback = {
  success: msg => message.success(msg),
  failed: msg => message.failed(msg),
};

export const someAction = (params) => ({
  types: [actionTypes.SOME_START, actionTypes.SOME_SUCCESS, actionTypes.SOME_FAILED],
  axiosAPI: () => axios({
    //一些参数
  }),
  callback: {
    // 消息提示
    messageSuccess: (msg = '请求成功！') => messageCallback.success(msg),
    messageFailed: (msg = '请求失败！') => messageCallback.failed(msg),
    // 对应错误码处理
    errorList: [{
      errorCode: 1010,
      errorMsg: '登录状态失效',
    }, {
      errorCode: 202,
      errorMsg: '信息重复',
    }],
    // 对后台返回数据处理
    handleData: (data) => {
    }
    // 函数回调
    callbackFunc: (dispatch, data) => {
      // 一些回调方法，如发起另外的action，data处理等
    }
  }
});
```

继续扩展我们的中间件：

```javascript
export default ({ dispatch, getState }) => next => action => {
  const { types, axiosAPI, callback = {} } = action;

  // 不存在types属性，则交由下个中间件处理
  if (!types) {
    return next(action);
  }
  
  // 验证types是否正确的数组
  if (!Array.isArray(types) || types.length !== 3 || !types.every(type => typeof type === 'string')) {
    throw new Error('Expected types to be an array of three string types.');
  }
  
  if (typeof axiosAPI !== 'function') {
    throw new Error('Expected axiosAPI to be a function.');
  }
  
  if (typeof callback !== 'object') {
    throw new Error('Expected callback to be an object.');
  }
  
  const [startType, successType, failureType] = types;
  
  // 因为callback和它的属性都为可选项，所以添加默认值
  const {
    messageSuccess = () => {},
    messageFailed = () => {},
    callbackFunc = () => {},
    errorList = [],
    handleData = () => {},
  } = callback;
  
  dispatch({ type: startType });
  
  return axiosAPI().then(response => {
    const { ErrCode, ErrMsg, data } = response.data;
    
    // 从errorList筛选出和后台返回的ErrCode对应的值
    const error = errorList.filter(item => item.errorCode === ErrCode)[0];
    let errorMsg = '';
    
    if (error) {
      errorMsg = error.errorMsg;
    }
    
    // 根据后台返回的错误码进行处理
    if (ErrCode !== 200) {
      if (error) {
        dispatch({ type: failureType, errorMsg });
      	return errorMsg && typeof errorMsg === 'string' ? messageFailed(errorMsg) : null;
      }
      
      dispatch({ type: failureType, errorMsg: ErrMsg || 'failed' });
      return messageFailed();
    } else if (ErrCode === 200) {
      dispatch({
        type: successType,
        data: handleData(data) || data,
      });
      callbackFunc(dispatch, data);
      return messageSuccess();
    }
  })
    .catch(error => {
      console.log(error);
      dispatch({ type: failureType, errorMsg: 'sometext'});
  })
}
```

大功告成！这样我们就可以编写比较清晰的异步请求action。

最后处理reducer：

```javascript
import * as actionTypes from '../actionTypes';

// 使用之前封装的createReducer
const initState = {
  isRequesting: false,
  data: [],
  errorMsg: '',
};
const someReducer = createReducer(initState, {
  [actionTypes.SOME_START](state, action) {
    return { ...state, isRequesting: true };
  },
  [actionTypes.SOME_SUCCESS](state, action) {
    return { ...state, data: action.data, isRequesting: false };
  },
  [actionTypes.SOME_FAILED](state, action) {
    return { ...state, errorMsg: action.errMsg, isRequesting: false };
  },
});
```

在编写reducer时我们会发现，异步请求的reducer往往具有相同的逻辑，即三种处理条件，我们试试将这些逻辑复用：

```javascript
const createAsyncReducer = (initState, types) => {
  if (!Array.isArray(types) || types.length !== 3 || !types.every(type => typeof type === 'string')) {
    throw new Error('Expected types to be an array of three string types.');
  }
  
  const [SOME_START, SOME_SUCCESS, SOME_FAILED] = types;
  
  return createReducer(initState, {
    [SOME_START](state, action) {
   	  return { ...state, isRequesting: true };
  	},
  	[SOME_SUCCESS](state, action) {
      return { ...state, data: action.data, isRequesting: false };
  	},
  	[SOME_FAILED](state, action) {
      return { ...state, errorMsg: action.errMsg, isRequesting: false };
  	},
  });
};
```

将其中的条件处理抽取出来，作为`createHandler`函数：

```javascript
const createHandler = types => {
  if (!Array.isArray(types) || types.length !== 3 || !types.every(type => typeof type === 'string')) {
    throw new Error('Expected types to be an array of three string types.');
  }
  
  const [SOME_START, SOME_SUCCESS, SOME_FAILED] = types;
  
  return {
    [SOME_START](state, action) {
   	  return { ...state, isRequesting: true };
  	},
  	[SOME_SUCCESS](state, action) {
      return { ...state, data: action.data, isRequesting: false };
  	},
  	[SOME_FAILED](state, action) {
      return { ...state, errorMsg: action.errMsg, isRequesting: false };
  	},
  };
};

// options为另外的reducer处理逻辑
const createAsyncReducer = (initState, types, options = {}) => (
  createReducer(initState, {
    ...createHandler(types),
    ...options,
  })
);
```

完成这些工具函数之后，编写异步请求的reducer只需要像下面这样：

```javascript
// 异步请求的reducer
import * as actionTypes from '../actionTypes';

const initState = {
  isRequesting: false,
  data: [],
  errorMsg: '',
  otherValue: '', // options要处理的数据
};
const options = {
  [actionTypes.OTHER_ACTION]: (state, action) => ({ ...state, otherValue: action.value }),
};

const types = [actionTypes.SOME_START, actionTypes.SOME_SUCCESS, actionTypes.SOME_FAILED];
const someRequestReducer = createAsyncReducer(initState, types, options);

// 在action/index.js里添加处理option的action
const otherAction = createAction(action.OTHER_ACTION, 'value');
```

## 以axios为基础再次扩展中间件

如果使用axios来调用AJAX，我们可以直接把axios写在中间件里，在编写异步action的时候添加axios的参数。

值得注意的是，在一个组件中调用dispatch时，我们往往会需要其他reducer的数据来作为参数，但我们又不需要使用这些数据来渲染组件，使用`getState`可以解决这个问题。

综合以上问题，再次扩展中间件：

```javascript
import axios from 'axios'; // 引入axios库

export default ({ dispatch, getState }) => next => action => {
  // params为axios参数
  const { types, generateParams, callback = {} } = action;

  if (!types) {
    return next(action);
  }
  
  if (!Array.isArray(types) || types.length !== 3 || !types.every(type => typeof type === 'string')) {
    throw new Error('Expected types to be an array of three string types.');
  }
  
  if (typeof generateParams !== 'function') {
    throw new Error('Expected generateParams to be a function.');
  }
  
  if (typeof callback !== 'object') {
    throw new Error('Expected callback to be an object.');
  }
  
  const [startType, successType, failureType] = types;
  const {
    messageSuccess = () => {},
    messageFailed = () => {},
    callbackFunc = () => {},
    errorList = [],
    handleData = () => {},
  } = callback;
  
  
  dispatch({ type: startType });
  
  // 传入getState作为参数
  return axios(generateParams(getState)).then(response => {
    const { ErrCode, ErrMsg, data } = response.data;
    const error = errorList.filter(item => item.errorCode === ErrCode)[0];
    let errorMsg = '';
    
    if (error) {
      errorMsg = error.errorMsg;
    }
    
    if (ErrCode !== 200) {
      if (error) {
        dispatch({ type: failureType, errorMsg });
      	return errorMsg && typeof errorMsg === 'string' ? messageFailed(errorMsg) : null;
      }
      
      dispatch({ type: failureType, errorMsg: ErrMsg || 'failed' });
      return messageFailed();
    } else if (ErrCode === 200) {
      dispatch({
        type: successType,
        data: handleData(data) || data,
      });
      callbackFunc(dispatch, data, getState);
      return messageSuccess();
    }
  })
    .catch(error => {
      console.log(error);
      dispatch({ type: failureType, errorMsg: 'sometext'});
  })
}
```

对应的action：

```typescript
import { message } from 'antd';

const messageCallback = {
  success: msg => message.success(msg),
  failed: msg => message.failed(msg),
};

export const someAction = (params) => ({
  types: [actionTypes.SOME_START, actionTypes.SOME_SUCCESS, actionTypes.SOME_FAILED],
  // 需要使用其他reducer，则使用getState
  generateParams: (getState) => {
    //...
  },
  // 否则直接省略参数，返回对象
  generateParams: () => ({
    //...
  })
  callback: {
    messageSuccess: (msg = '请求成功！') => messageCallback.success(msg),
    messageFailed: (msg = '请求失败！') => messageCallback.failed(msg),
    errorList: [{
      errorCode: 1010,
      errorMsg: '登录状态失效',
    }, {
      errorCode: 202,
      errorMsg: '信息重复',
    }],
    handleData: (data) => {
    }
    callbackFunc: (dispatch, data, getState) => {
      // 一些回调方法，如发起另外的action，data处理等
    }
  }
});
```

最后要说的是，因为不同的项目有不同的需求，可能上面的工具函数无法与你当前的需求完全匹配，但是思想是一致的。因为redux的设计原则导致它容易编写出大量类似的代码，这时你可以考虑像上面一样，封装自己的工具函数。





