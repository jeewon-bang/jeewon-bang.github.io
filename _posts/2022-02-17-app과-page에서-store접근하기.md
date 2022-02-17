---
layout: post
title: "_app과 page에서 store 접근하기"
---

<br />

<aside>
✨ 페이지의 `getInitialProps`를 사용하면 페이지를 그리는동안 Redux store에 접근할 수 있음. `next-redux-wrapper`를 사용하면 자동으로 store인스턴스를 만들어줌

</aside>

<br />

### ⚙️Installation

```jsx
npm install next-redux-wrapper react-redux --save
```

<br />

### 🗃️ store만들기

```jsx
// store.ts

import { createStore, AnyAction, Store } from "redux";
import { createWrapper, Context, HYDRATE } from "next-redux-wrapper";

export interface State {
  tick: string;
}

// create your reducer
const reducer = (state: State = { tick: "init" }, action: AnyAction) => {
  switch (action.type) {
    case HYDRATE:
      // Attention! This will overwrite client state! Real apps should use proper reconciliation.
      return { ...state, ...action.payload };
    case "TICK":
      return { ...state, tick: action.payload };
    default:
      return state;
  }
};

// create a makeStore function
const makeStore = (context: Context) => createStore(reducer);

// export an assembled wrapper
export const wrapper =
  createWrapper < Store < State >> (makeStore, { debug: true });
```

- `createWrapper`함수는 첫번째 인자로 `makeStore`함수를 받음
- `makeStore`함수는 그것이 불려질 때마다 새로운 redux store 인스턴스를 리턴함
- 두번째 인자는 config객체를 받음
- `{debug: true}` : debug로그 정보를 콘솔에 출력함

<br />

### ✅ 페이지에서 getServerSideProps 사용

```jsx
import React from "react";
import { NextPage } from "next";
import { connect } from "react-redux";
import { wrapper, State } from "../store";

export const getServerSideProps = wrapper.getServerSideProps(
  (store) =>
    ({ req, res, ...etc }) => {
      console.log(
        "2. Page.getServerSideProps uses the store to dispatch things"
      );
      store.dispatch({ type: "TICK", payload: "was set in other page" });
    }
);

// Page itself is not connected to Redux Store, it has to render Provider to allow child components to connect to Redux Store
const Page: NextPage<State> = ({ tick }) => <div>{tick}</div>;

// you can also use Redux `useSelector` and other hooks instead of `connect()`
export default connect((state: State) => state)(Page);
```

- `pages/pageName.tsx`에서 사용

<br />

### ✅ 페이지에서 getInitialProps 사용

```jsx
import React, {Component} from 'react';
import {NextPage} from 'next';
import {wrapper, State} from '../store';

// you can also use `connect()` instead of hooks
const Page: NextPage = () => {
  const {tick} = useSelector<State, State>(state => state);
  return <div>{tick}</div>;
};

Page.getInitialProps = wrapper.getInitialPageProps(store => ({pathname, req, res}) => {
  console.log('2. Page.getInitialProps uses the store to dispatch things');
  store.dispatch({
    type: 'TICK',
    payload: 'was set in error page ' + pathname,
  });
});

export default Page;
```

- page에서 `getInitialProps`를 사용하려면 `wrapper`의 `getInitialPageProps`를 대입?해야함
- 클라이언트 사이드에서 `getInitialProps`가 불려지면 `req`와 `res`를 사용할 수 없음

<br />

### ✅ App에서 getInitialProps

```jsx
# pages/_app.tsx

import React from 'react';
import App, {AppInitialProps} from 'next/app';
import {wrapper} from '../components/store';
import {State} from '../components/reducer';

// Since you'll be passing more stuff to Page
declare module 'next/dist/next-server/lib/utils' {
    export interface NextPageContext {
        store: Store<State>;
    }
}

class MyApp extends App<AppInitialProps> {

    public static getInitialProps = wrapper.getInitialAppProps(store => async context => {

        store.dispatch({type: 'TOE', payload: 'was set in _app'});

        return {
            pageProps: {
                // https://nextjs.org/docs/advanced-features/custom-app#caveats
                ...(await App.getInitialProps(context)).pageProps,
                // Some custom thing for all pages
                pathname: ctx.pathname,
            },
        };

    });

    public render() {
        const {Component, pageProps} = this.props;

        return (
            <Component {...pageProps} />
        );
    }
}

export default wrapper.withRedux(MyApp);
```

> You can dispatch actions from the `pages/_app` too. But this mode is not compatible with [Next.js 9's Auto Partial Static Export](https://nextjs.org/blog/next-9#automatic-partial-static-export) feature, see the [explanation below](https://github.com/kirill-konshin/next-redux-wrapper#automatic-partial-static-export).

- ⚠️ `pages/_app`에서도 액션을 dispatch할 수 있음
- 그런데 이것은 next.js 9와 호환이 안 됨

<br />

### 🚦 How it works

`next-redux-wrapper`(”the wrapper”)를 사용하면 다음 request이 생김

- Phase 1: getInitialProps/getStaticProps/getServerSideProps
  - wrapper는 비어있는 initial state로 **server쪽의 store**를 만듦. `makeStore`로 Request와 Response객체도 제공됨?
  - In App mode:
    - wrapper는 `_app`의 `getInitialProps`를 호출하고 이전에 만들어진 store를 전달함
    - Next.js는 그 `props`를 받아서 store의 state와 함께 리턴함
  - In per-page mode:
    - wrapper는 page의 `getXXXProps`를 호출하고 이전에 만들어진 store를 전달함
    - Next.js는 그 `props`를 받아서 store의 state와 함께 리턴함
- Phase 2: SSR
  - wrapper는 `makeStore`를 통해 새로운 store를 만듦
  - wrapper는 이전의 store의 state인 payload와 함께 `HYDRATE`를 보냄(dispatch)
  - 그 store는 `_app`이나 `page`컴포넌트에 속성(property)로 보내짐
  - **연결되어있는 컴포넌트들은 store의 state를 바꾸지만, 변경된 state는 client에 전달되지 않음**
- Phase 3: Client
  - wrapper는 새로운 store를 만듦
  - wrapper는 Phase 1의 state로부터 payload를 통해 `HYDRATE`를 보냄(dispatch)
  - 그 store는 `_app`이나 `page`컴포넌트의 속성(property)로 보내짐
  - **wrapper는 client의 `window`객체에 그 store를 저장하고, 그러면 HMR??일 경우 restore됨????**
