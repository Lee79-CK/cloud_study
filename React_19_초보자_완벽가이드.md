# 🚀 React 19 완벽 가이드 - 웹 개발 초보자를 위한 친절한 강의

> 본 강의는 React 19.2 (2026년 최신 버전)를 기준으로 작성되었습니다.
> 프로그래밍을 처음 시작하는 분들도 차근차근 따라올 수 있도록 구성했습니다.

---

## 📚 목차

1. [React란 무엇인가?](#1-react란-무엇인가)
2. [개발 환경 설정](#2-개발-환경-설정)
3. [JSX - HTML과 비슷하지만 더 강력한 문법](#3-jsx---html과-비슷하지만-더-강력한-문법)
4. [컴포넌트 - React의 핵심 단위](#4-컴포넌트---react의-핵심-단위)
5. [Props - 컴포넌트에 데이터 전달하기](#5-props---컴포넌트에-데이터-전달하기)
6. [State와 useState - 변하는 데이터 다루기](#6-state와-usestate---변하는-데이터-다루기)
7. [이벤트 핸들링 - 사용자와 상호작용하기](#7-이벤트-핸들링---사용자와-상호작용하기)
8. [조건부 렌더링과 리스트 렌더링](#8-조건부-렌더링과-리스트-렌더링)
9. [useEffect - 부수 효과 다루기](#9-useeffect---부수-효과-다루기)
10. [React 19의 새로운 기능들](#10-react-19의-새로운-기능들)
11. [실전 프로젝트: 할 일 관리 앱 만들기](#11-실전-프로젝트-할-일-관리-앱-만들기)
12. [React 활용 방안과 다음 단계](#12-react-활용-방안과-다음-단계)

---

## 1. React란 무엇인가?

### 🤔 React를 한 마디로 정의하면?

**React는 "사용자 인터페이스(UI)를 만들기 위한 JavaScript 라이브러리"입니다.**

좀 더 쉽게 말하면, 웹사이트의 화면을 효율적으로 만들 수 있게 도와주는 도구입니다. Facebook(현 Meta)에서 2013년에 처음 만들었고, 현재는 전 세계에서 가장 많이 사용되는 프론트엔드 도구입니다.

### 🏠 React를 비유로 이해하기

집을 짓는다고 상상해봅시다.

- **HTML만 사용** = 모든 벽돌을 일일이 쌓아가며 집을 짓는 것
- **React 사용** = 미리 만들어진 모듈(방, 화장실, 주방)을 조립해 집을 짓는 것

React에서는 화면의 각 부분을 **"컴포넌트(Component)"**라는 작은 조각으로 만들고, 이 조각들을 조립해서 전체 화면을 구성합니다.

### ✨ React의 특징

1. **컴포넌트 기반(Component-Based)**
   - 화면을 작은 부품으로 나눠서 만들고 재사용할 수 있습니다.

2. **선언적 UI(Declarative)**
   - "어떻게 그릴지"가 아니라 "무엇을 그릴지"만 알려주면 됩니다.

3. **가상 DOM(Virtual DOM)**
   - 실제 화면을 직접 바꾸지 않고, 가상의 화면에서 먼저 계산한 후 변경된 부분만 업데이트해서 빠릅니다.

4. **풍부한 생태계**
   - 전 세계 개발자들이 만든 수많은 라이브러리를 활용할 수 있습니다.

### 🌍 어디에 쓰이나?

- **Instagram**, **Facebook** - 소셜 미디어
- **Netflix** - 동영상 스트리밍
- **Airbnb** - 숙박 예약
- **Discord** - 메신저
- **WhatsApp Web** - 웹 메신저
- 한국의 **토스**, **당근마켓**, **배달의민족** 등 수많은 서비스

---

## 2. 개발 환경 설정

### 🛠️ 필요한 도구

React 개발을 시작하려면 두 가지가 필요합니다.

#### 1️⃣ Node.js 설치

Node.js는 컴퓨터에서 JavaScript를 실행할 수 있게 해주는 프로그램입니다.

- [Node.js 공식 사이트](https://nodejs.org/)에서 LTS 버전을 다운로드 받아 설치하세요.
- 설치 후 터미널(Mac) 또는 명령 프롬프트(Windows)에서 확인:

```bash
node -v
npm -v
```

버전 번호가 출력되면 성공입니다.

#### 2️⃣ 코드 에디터

**Visual Studio Code(VS Code)**를 추천합니다. 무료이며 React 개발에 최적화되어 있습니다.
- [VS Code 다운로드](https://code.visualstudio.com/)

추천 확장 프로그램:
- **ES7+ React/Redux/React-Native snippets** - 코드 자동 완성
- **Prettier** - 코드 자동 정렬
- **Auto Rename Tag** - HTML 태그 자동 이름 변경

### 🚀 첫 React 프로젝트 만들기 (Vite 사용)

요즘은 **Vite(비트)**라는 빌드 도구로 React 프로젝트를 만드는 것이 표준입니다. 매우 빠르고 가볍습니다.

터미널을 열고 다음 명령어를 입력하세요:

```bash
# my-first-react-app 이라는 이름으로 React 프로젝트 생성
npm create vite@latest my-first-react-app -- --template react

# 프로젝트 폴더로 이동
cd my-first-react-app

# 필요한 패키지 설치
npm install

# 개발 서버 실행
npm run dev
```

브라우저에서 `http://localhost:5173`을 열면 React가 동작하는 것을 볼 수 있습니다! 🎉

### 📁 프로젝트 폴더 구조 살펴보기

```
my-first-react-app/
├── node_modules/      # 설치된 라이브러리들 (건드리지 않음)
├── public/            # 정적 파일 (이미지, 아이콘 등)
├── src/               # 우리가 코드를 작성하는 폴더 ⭐
│   ├── App.jsx        # 메인 컴포넌트
│   ├── main.jsx       # 시작 지점
│   └── index.css      # 스타일
├── index.html         # HTML 파일
├── package.json       # 프로젝트 정보 및 의존성
└── vite.config.js     # Vite 설정
```

가장 중요한 폴더는 **`src`**입니다. 여기서 모든 코드를 작성하게 됩니다.

---

## 3. JSX - HTML과 비슷하지만 더 강력한 문법

### 📝 JSX란?

JSX(JavaScript XML)는 **JavaScript 안에서 HTML처럼 생긴 코드를 작성할 수 있게 해주는 문법**입니다.

```jsx
// 이것이 JSX입니다!
const element = <h1>안녕하세요, React!</h1>;
```

처음 보면 이상해 보일 수 있지만, 익숙해지면 매우 직관적입니다.

### 🎨 JSX 기본 규칙

#### 1) 반드시 하나의 부모 요소로 감싸야 합니다

```jsx
// ❌ 잘못된 예
return (
  <h1>제목</h1>
  <p>내용</p>
);

// ✅ 올바른 예 - div로 감싸기
return (
  <div>
    <h1>제목</h1>
    <p>내용</p>
  </div>
);

// ✅ Fragment 사용 (불필요한 div를 만들지 않음)
return (
  <>
    <h1>제목</h1>
    <p>내용</p>
  </>
);
```

#### 2) 모든 태그를 닫아야 합니다

```jsx
// ❌ 잘못된 예
<img src="photo.jpg">
<br>

// ✅ 올바른 예
<img src="photo.jpg" />
<br />
```

#### 3) JavaScript 표현식은 `{}` 안에 작성합니다

```jsx
const name = "철수";
const age = 25;

return (
  <div>
    <h1>안녕하세요, {name}님!</h1>
    <p>나이: {age}살</p>
    <p>내년 나이: {age + 1}살</p>
    <p>현재 시간: {new Date().toLocaleTimeString()}</p>
  </div>
);
```

#### 4) `class` 대신 `className`을 사용합니다

JavaScript에서 `class`는 예약어이기 때문입니다.

```jsx
// ❌ HTML 방식
<div class="container">

// ✅ JSX 방식
<div className="container">
```

#### 5) 인라인 스타일은 객체로 작성합니다

```jsx
// CSS 속성명은 camelCase로!
const style = {
  backgroundColor: 'lightblue',  // background-color ❌
  fontSize: '20px',              // font-size ❌
  padding: '10px'
};

return <div style={style}>스타일이 적용된 박스</div>;

// 또는 인라인으로 직접
return <div style={{ color: 'red', fontSize: '24px' }}>빨간 글씨</div>;
```

---

## 4. 컴포넌트 - React의 핵심 단위

### 🧩 컴포넌트란?

**컴포넌트는 화면의 한 부분을 담당하는 재사용 가능한 코드 조각입니다.**

레고 블록을 떠올려보세요. 작은 블록들을 조립해 큰 작품을 만드는 것처럼, React에서는 작은 컴포넌트들을 조립해 큰 애플리케이션을 만듭니다.

### 🏗️ 함수형 컴포넌트 만들기

React에서는 **함수형 컴포넌트**가 표준입니다.

```jsx
// Welcome.jsx 파일
function Welcome() {
  return <h1>환영합니다!</h1>;
}

export default Welcome;
```

**컴포넌트 작성 규칙:**

1. 컴포넌트 이름은 **반드시 대문자로 시작**합니다 (예: `Welcome`, `MyButton`).
2. 함수가 JSX를 `return`해야 합니다.
3. 다른 파일에서 사용하려면 `export`해야 합니다.

### 📥 컴포넌트 사용하기

```jsx
// App.jsx
import Welcome from './Welcome';

function App() {
  return (
    <div>
      <Welcome />
      <Welcome />  {/* 재사용 가능! */}
      <Welcome />
    </div>
  );
}

export default App;
```

### 🎯 실습: 간단한 카드 컴포넌트 만들기

```jsx
// ProfileCard.jsx
function ProfileCard() {
  return (
    <div style={{
      border: '1px solid #ddd',
      borderRadius: '8px',
      padding: '20px',
      width: '300px',
      boxShadow: '0 2px 4px rgba(0,0,0,0.1)'
    }}>
      <h2>홍길동</h2>
      <p>📧 hong@example.com</p>
      <p>📱 010-1234-5678</p>
      <p>💼 프론트엔드 개발자</p>
    </div>
  );
}

export default ProfileCard;
```

이 컴포넌트는 어디서든 `<ProfileCard />`로 사용할 수 있습니다. 하지만 항상 같은 사람의 정보만 표시되겠죠? 다음 단원에서 이 문제를 해결하는 **Props**를 배워봅시다.

---

## 5. Props - 컴포넌트에 데이터 전달하기

### 📦 Props란?

**Props(properties의 줄임말)는 부모 컴포넌트가 자식 컴포넌트에 전달하는 데이터입니다.**

함수에 매개변수(parameter)를 전달하는 것과 비슷합니다.

```jsx
// 함수에 매개변수 전달
function greet(name) {
  return `안녕하세요, ${name}님!`;
}
greet("철수");

// 컴포넌트에 props 전달
function Greet({ name }) {
  return <p>안녕하세요, {name}님!</p>;
}
<Greet name="철수" />
```

### 🎁 Props 사용 예제

```jsx
// ProfileCard.jsx
function ProfileCard({ name, email, phone, job }) {
  return (
    <div style={{
      border: '1px solid #ddd',
      borderRadius: '8px',
      padding: '20px',
      width: '300px',
      margin: '10px'
    }}>
      <h2>{name}</h2>
      <p>📧 {email}</p>
      <p>📱 {phone}</p>
      <p>💼 {job}</p>
    </div>
  );
}

export default ProfileCard;
```

```jsx
// App.jsx
import ProfileCard from './ProfileCard';

function App() {
  return (
    <div>
      <ProfileCard 
        name="홍길동"
        email="hong@example.com"
        phone="010-1111-1111"
        job="프론트엔드 개발자"
      />
      <ProfileCard 
        name="김영희"
        email="kim@example.com"
        phone="010-2222-2222"
        job="백엔드 개발자"
      />
      <ProfileCard 
        name="이철수"
        email="lee@example.com"
        phone="010-3333-3333"
        job="디자이너"
      />
    </div>
  );
}
```

### 💡 Props는 읽기 전용입니다

자식 컴포넌트에서 부모로부터 받은 props를 직접 수정해서는 안 됩니다.

```jsx
function Bad({ name }) {
  name = "변경됨";  // ❌ 절대 안 됨!
  return <p>{name}</p>;
}
```

### 🎁 Props에 기본값 설정하기

```jsx
function Greeting({ name = "방문자" }) {
  return <h1>안녕하세요, {name}님!</h1>;
}

<Greeting />              // "안녕하세요, 방문자님!"
<Greeting name="철수" />   // "안녕하세요, 철수님!"
```

### 🎯 children prop - 특별한 prop

`children`은 컴포넌트 태그 사이에 들어가는 내용을 받는 특별한 prop입니다.

```jsx
function Card({ children }) {
  return (
    <div style={{ border: '1px solid #ddd', padding: '20px' }}>
      {children}
    </div>
  );
}

// 사용
<Card>
  <h2>제목</h2>
  <p>내용입니다.</p>
</Card>
```

---

## 6. State와 useState - 변하는 데이터 다루기

### 🔄 State란?

**State는 컴포넌트 내부에서 관리되는 "변하는 데이터"입니다.**

- **Props**: 부모로부터 받는 데이터 (변경 불가)
- **State**: 컴포넌트가 자체적으로 관리하는 데이터 (변경 가능)

예: 좋아요 버튼의 클릭 횟수, 입력 폼의 텍스트, 모달 열림/닫힘 상태 등

### 🪝 useState - 첫 번째 Hook

`useState`는 React에서 제공하는 **Hook(훅)**입니다. Hook은 함수형 컴포넌트에 특별한 기능을 추가해주는 도구입니다.

```jsx
import { useState } from 'react';

function Counter() {
  // [현재 값, 값을 변경하는 함수] = useState(초기값)
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>현재 카운트: {count}</p>
      <button onClick={() => setCount(count + 1)}>
        +1 증가
      </button>
      <button onClick={() => setCount(count - 1)}>
        -1 감소
      </button>
      <button onClick={() => setCount(0)}>
        초기화
      </button>
    </div>
  );
}
```

### 📚 useState 작동 원리

```jsx
const [count, setCount] = useState(0);
//      ↑       ↑                ↑
//   현재값   변경함수         초기값
```

1. 처음 렌더링 시 `count`는 `0`이 됩니다.
2. `setCount(5)`를 호출하면 `count`가 `5`로 변경됩니다.
3. **state가 변경되면 컴포넌트가 자동으로 다시 렌더링됩니다!** (이게 React의 핵심)

### ⚠️ 중요한 규칙

#### 1) state를 직접 수정하지 마세요

```jsx
// ❌ 절대 안 됨!
count = count + 1;

// ✅ 반드시 setter 함수 사용
setCount(count + 1);
```

#### 2) 객체나 배열 state는 새로운 것을 만들어 전달하세요

```jsx
const [user, setUser] = useState({ name: "철수", age: 25 });

// ❌ 잘못된 방법
user.age = 26;
setUser(user);

// ✅ 올바른 방법 - 새 객체 생성
setUser({ ...user, age: 26 });
```

```jsx
const [items, setItems] = useState([1, 2, 3]);

// ❌ 잘못된 방법
items.push(4);
setItems(items);

// ✅ 올바른 방법 - 새 배열 생성
setItems([...items, 4]);
```

### 🎨 다양한 state 예제

```jsx
import { useState } from 'react';

function MultipleStates() {
  // 문자열 state
  const [name, setName] = useState('');
  
  // 불리언 state
  const [isVisible, setIsVisible] = useState(true);
  
  // 객체 state
  const [user, setUser] = useState({ 
    nickname: '익명', 
    level: 1 
  });
  
  // 배열 state
  const [todos, setTodos] = useState([]);
  
  return (
    <div>
      <input 
        type="text" 
        value={name} 
        onChange={(e) => setName(e.target.value)} 
        placeholder="이름을 입력하세요"
      />
      <p>입력된 이름: {name}</p>
      
      <button onClick={() => setIsVisible(!isVisible)}>
        {isVisible ? '숨기기' : '보이기'}
      </button>
      {isVisible && <p>이 텍스트는 보이거나 숨겨집니다.</p>}
      
      <p>닉네임: {user.nickname}, 레벨: {user.level}</p>
      <button onClick={() => setUser({ ...user, level: user.level + 1 })}>
        레벨업!
      </button>
    </div>
  );
}
```

---

## 7. 이벤트 핸들링 - 사용자와 상호작용하기

### 🖱️ 이벤트란?

이벤트는 사용자가 페이지에서 일으키는 동작입니다.
- 클릭 (click)
- 입력 (change, input)
- 마우스 이동 (mouseover)
- 키보드 입력 (keypress)
- 폼 제출 (submit)

### 📌 React에서 이벤트 처리하기

HTML에서는 `onclick`처럼 소문자로 쓰지만, React에서는 **camelCase**로 작성합니다.

```jsx
// HTML
<button onclick="handleClick()">클릭</button>

// React (JSX)
<button onClick={handleClick}>클릭</button>
```

### 🎯 다양한 이벤트 예제

```jsx
import { useState } from 'react';

function EventExample() {
  const [message, setMessage] = useState('');

  // 클릭 이벤트
  const handleClick = () => {
    alert('버튼이 클릭되었습니다!');
  };

  // 입력 이벤트
  const handleChange = (event) => {
    setMessage(event.target.value);
  };

  // 폼 제출 이벤트
  const handleSubmit = (event) => {
    event.preventDefault();  // 기본 새로고침 방지
    alert(`제출된 메시지: ${message}`);
  };

  // 마우스 이벤트
  const handleMouseOver = () => {
    console.log('마우스가 올라왔습니다');
  };

  return (
    <div>
      <button onClick={handleClick}>클릭하세요</button>
      
      <button onMouseOver={handleMouseOver}>
        마우스를 올려보세요
      </button>
      
      <form onSubmit={handleSubmit}>
        <input 
          type="text" 
          value={message} 
          onChange={handleChange}
          placeholder="메시지 입력"
        />
        <button type="submit">전송</button>
      </form>
    </div>
  );
}
```

### 🎁 이벤트에 인자(argument) 전달하기

```jsx
function ButtonList() {
  const handleClick = (item) => {
    alert(`${item}을(를) 선택했습니다!`);
  };

  return (
    <div>
      {/* 화살표 함수로 감싸서 전달 */}
      <button onClick={() => handleClick('사과')}>사과</button>
      <button onClick={() => handleClick('바나나')}>바나나</button>
      <button onClick={() => handleClick('포도')}>포도</button>
    </div>
  );
}
```

---

## 8. 조건부 렌더링과 리스트 렌더링

### 🔀 조건부 렌더링

상황에 따라 다른 화면을 보여주는 방법입니다.

#### 방법 1: if문 사용

```jsx
function Greeting({ isLoggedIn }) {
  if (isLoggedIn) {
    return <h1>환영합니다!</h1>;
  } else {
    return <h1>로그인해주세요.</h1>;
  }
}
```

#### 방법 2: 삼항 연산자 (가장 흔히 사용)

```jsx
function Greeting({ isLoggedIn }) {
  return (
    <div>
      {isLoggedIn ? <h1>환영합니다!</h1> : <h1>로그인해주세요.</h1>}
    </div>
  );
}
```

#### 방법 3: && 연산자 (한쪽만 표시할 때)

```jsx
function Notification({ hasMessage }) {
  return (
    <div>
      <h1>알림</h1>
      {hasMessage && <p>새 메시지가 있습니다!</p>}
    </div>
  );
}
```

### 📋 리스트 렌더링 - map() 함수 사용

배열 데이터를 화면에 표시할 때는 JavaScript의 `map()` 함수를 사용합니다.

```jsx
function FruitList() {
  const fruits = ['사과', '바나나', '포도', '딸기', '오렌지'];

  return (
    <ul>
      {fruits.map((fruit, index) => (
        <li key={index}>{fruit}</li>
      ))}
    </ul>
  );
}
```

### 🔑 key prop의 중요성

**리스트의 각 항목에는 반드시 `key`를 지정해야 합니다!** key는 React가 어떤 항목이 변경, 추가, 삭제되었는지 식별하는 데 사용됩니다.

```jsx
function UserList() {
  const users = [
    { id: 1, name: '홍길동', age: 25 },
    { id: 2, name: '김영희', age: 30 },
    { id: 3, name: '이철수', age: 28 }
  ];

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>
          {user.name} ({user.age}세)
        </li>
      ))}
    </ul>
  );
}
```

> 💡 **key는 가능하면 고유한 ID를 사용하세요.** 배열의 index를 key로 사용하는 것은 가급적 피해야 합니다 (특히 항목이 추가/삭제/순서 변경되는 경우).

### 🎨 종합 예제: 상품 목록

```jsx
function ProductList() {
  const products = [
    { id: 1, name: '노트북', price: 1500000, inStock: true },
    { id: 2, name: '마우스', price: 30000, inStock: false },
    { id: 3, name: '키보드', price: 80000, inStock: true },
    { id: 4, name: '모니터', price: 350000, inStock: true }
  ];

  return (
    <div>
      <h1>🛒 상품 목록</h1>
      {products.length === 0 ? (
        <p>등록된 상품이 없습니다.</p>
      ) : (
        <ul>
          {products.map((product) => (
            <li key={product.id} style={{ marginBottom: '10px' }}>
              <strong>{product.name}</strong> - 
              {product.price.toLocaleString()}원
              {product.inStock ? (
                <span style={{ color: 'green' }}> ✅ 재고 있음</span>
              ) : (
                <span style={{ color: 'red' }}> ❌ 품절</span>
              )}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

---

## 9. useEffect - 부수 효과 다루기

### 🌊 부수 효과(Side Effect)란?

컴포넌트가 렌더링된 후에 수행해야 하는 작업들입니다.

- 서버에서 데이터 가져오기 (API 호출)
- 브라우저 제목 변경
- 타이머 설정
- 이벤트 리스너 등록/해제
- 외부 라이브러리 연결

### 🪝 useEffect 기본 사용법

```jsx
import { useState, useEffect } from 'react';

function Clock() {
  const [time, setTime] = useState(new Date());

  useEffect(() => {
    // 1초마다 시간 업데이트
    const timer = setInterval(() => {
      setTime(new Date());
    }, 1000);

    // 정리(cleanup) 함수: 컴포넌트가 사라질 때 실행됨
    return () => {
      clearInterval(timer);
    };
  }, []);  // 빈 배열: 컴포넌트가 처음 렌더링될 때만 실행

  return <h1>현재 시간: {time.toLocaleTimeString()}</h1>;
}
```

### 📋 useEffect의 3가지 패턴

#### 1) 한 번만 실행 (마운트 시)

```jsx
useEffect(() => {
  console.log('컴포넌트가 처음 나타났습니다');
}, []);  // 빈 의존성 배열
```

#### 2) 매 렌더링마다 실행

```jsx
useEffect(() => {
  console.log('렌더링되었습니다');
});  // 의존성 배열 없음
```

#### 3) 특정 값이 바뀔 때만 실행

```jsx
useEffect(() => {
  console.log(`count가 ${count}로 변경되었습니다`);
}, [count]);  // count가 바뀔 때만 실행
```

### 🌐 API 데이터 가져오기 예제

```jsx
import { useState, useEffect } from 'react';

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    setLoading(true);
    
    fetch(`https://jsonplaceholder.typicode.com/users/${userId}`)
      .then(response => response.json())
      .then(data => {
        setUser(data);
        setLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setLoading(false);
      });
  }, [userId]);  // userId가 바뀌면 다시 호출

  if (loading) return <p>⏳ 로딩 중...</p>;
  if (error) return <p>❌ 에러: {error}</p>;
  if (!user) return null;

  return (
    <div>
      <h2>{user.name}</h2>
      <p>📧 {user.email}</p>
      <p>📱 {user.phone}</p>
    </div>
  );
}
```

---

## 10. React 19의 새로운 기능들

React 19는 2024년 12월에 정식 출시되었고, 19.2 버전이 2025년 10월에 나왔습니다. 많은 혁신적인 기능들이 추가되었어요.

### ⚡ 1) Actions - 폼 처리가 훨씬 쉬워졌어요

기존에는 폼 제출 시 로딩, 에러, 성공 상태를 일일이 관리해야 했습니다. React 19의 **Actions**는 이를 자동으로 처리해줍니다.

```jsx
import { useActionState } from 'react';

async function submitForm(previousState, formData) {
  const name = formData.get('name');
  
  try {
    // 서버에 데이터 전송
    await fetch('/api/users', {
      method: 'POST',
      body: JSON.stringify({ name })
    });
    return { success: true, message: '저장되었습니다!' };
  } catch (error) {
    return { success: false, message: '오류가 발생했습니다.' };
  }
}

function MyForm() {
  const [state, formAction, isPending] = useActionState(submitForm, null);

  return (
    <form action={formAction}>
      <input name="name" placeholder="이름" />
      <button type="submit" disabled={isPending}>
        {isPending ? '저장 중...' : '저장'}
      </button>
      {state?.message && <p>{state.message}</p>}
    </form>
  );
}
```

### 🎯 2) useOptimistic - 즉각적인 UI 반응

서버 응답을 기다리지 않고 미리 UI를 업데이트하는 기능입니다. 사용자 경험이 훨씬 좋아져요!

```jsx
import { useOptimistic, useState } from 'react';

function MessageList() {
  const [messages, setMessages] = useState([
    { id: 1, text: '안녕하세요!' }
  ]);
  
  const [optimisticMessages, addOptimisticMessage] = useOptimistic(
    messages,
    (current, newMessage) => [...current, { id: 'temp', text: newMessage, sending: true }]
  );

  async function sendMessage(formData) {
    const text = formData.get('message');
    addOptimisticMessage(text);  // 즉시 UI에 표시
    
    // 실제로 서버에 전송
    const sent = await fakeServerSend(text);
    setMessages([...messages, sent]);
  }

  return (
    <div>
      {optimisticMessages.map(msg => (
        <p key={msg.id}>
          {msg.text} {msg.sending && '(전송 중...)'}
        </p>
      ))}
      <form action={sendMessage}>
        <input name="message" />
        <button type="submit">전송</button>
      </form>
    </div>
  );
}
```

### 🔮 3) use() API - Promise를 직접 사용

`use()`는 Promise나 Context를 컴포넌트 안에서 직접 읽을 수 있게 해줍니다.

```jsx
import { use, Suspense } from 'react';

function UserName({ userPromise }) {
  // Promise를 직접 use()로 풀어쓰기
  const user = use(userPromise);
  return <h1>{user.name}</h1>;
}

function App() {
  const userPromise = fetchUser();
  
  return (
    <Suspense fallback={<p>로딩 중...</p>}>
      <UserName userPromise={userPromise} />
    </Suspense>
  );
}
```

### 🎬 4) Activity - 화면 상태 보존하기 (React 19.2)

탭 전환 시 컴포넌트 상태를 보존하는 새로운 API입니다.

```jsx
import { Activity } from 'react';

function App({ activeTab }) {
  return (
    <>
      <Activity mode={activeTab === 'home' ? 'visible' : 'hidden'}>
        <HomePage />
      </Activity>
      <Activity mode={activeTab === 'profile' ? 'visible' : 'hidden'}>
        <ProfilePage />
      </Activity>
    </>
  );
}
```

### 🪝 5) useEffectEvent - 더 깔끔한 useEffect

이펙트에서 반응적이지 않은 로직을 분리할 수 있습니다.

```jsx
import { useEffect, useEffectEvent } from 'react';

function ChatRoom({ roomId, theme }) {
  const onConnected = useEffectEvent(() => {
    showNotification(`${roomId}에 연결됨`, theme);
  });

  useEffect(() => {
    const connection = createConnection(roomId);
    connection.on('connected', () => {
      onConnected();  // theme이 바뀌어도 이펙트가 다시 실행되지 않음
    });
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]);  // roomId만 의존성에 포함
}
```

### 🚀 6) React Server Components (RSC)

서버에서 컴포넌트를 렌더링해 초기 로딩을 빠르게 만드는 기능입니다. Next.js 등 프레임워크에서 본격적으로 사용됩니다.

```jsx
// 'use server' 디렉티브로 서버 함수 표시
'use server';

async function deletePost(id) {
  await db.posts.delete(id);
}

// 서버에서만 실행되는 컴포넌트
async function PostList() {
  const posts = await db.posts.findMany();  // DB에서 직접 조회
  
  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### 📌 7) ref가 prop으로 전달 가능

`forwardRef` 없이도 ref를 그냥 prop처럼 전달할 수 있습니다.

```jsx
// React 19 이전
const MyInput = forwardRef((props, ref) => {
  return <input ref={ref} {...props} />;
});

// React 19
function MyInput({ ref, ...props }) {
  return <input ref={ref} {...props} />;
}
```

---

## 11. 실전 프로젝트: 할 일 관리 앱 만들기

지금까지 배운 내용을 종합한 **Todo 앱**을 만들어봅시다!

### 📝 전체 코드

```jsx
// TodoApp.jsx
import { useState } from 'react';

function TodoApp() {
  const [todos, setTodos] = useState([]);
  const [input, setInput] = useState('');
  const [filter, setFilter] = useState('all');  // all, active, completed

  // 할 일 추가
  const addTodo = (e) => {
    e.preventDefault();
    if (input.trim() === '') return;
    
    const newTodo = {
      id: Date.now(),
      text: input,
      completed: false
    };
    
    setTodos([...todos, newTodo]);
    setInput('');
  };

  // 완료 상태 토글
  const toggleTodo = (id) => {
    setTodos(todos.map(todo =>
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    ));
  };

  // 할 일 삭제
  const deleteTodo = (id) => {
    setTodos(todos.filter(todo => todo.id !== id));
  };

  // 필터링된 할 일 목록
  const filteredTodos = todos.filter(todo => {
    if (filter === 'active') return !todo.completed;
    if (filter === 'completed') return todo.completed;
    return true;
  });

  // 통계
  const totalCount = todos.length;
  const completedCount = todos.filter(t => t.completed).length;
  const activeCount = totalCount - completedCount;

  return (
    <div style={{
      maxWidth: '500px',
      margin: '50px auto',
      padding: '30px',
      backgroundColor: '#f9f9f9',
      borderRadius: '12px',
      boxShadow: '0 4px 12px rgba(0,0,0,0.1)',
      fontFamily: 'sans-serif'
    }}>
      <h1 style={{ textAlign: 'center', color: '#333' }}>
        📝 할 일 관리
      </h1>

      {/* 입력 폼 */}
      <form onSubmit={addTodo} style={{ display: 'flex', gap: '10px', marginBottom: '20px' }}>
        <input
          type="text"
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="할 일을 입력하세요..."
          style={{
            flex: 1,
            padding: '10px',
            border: '1px solid #ddd',
            borderRadius: '6px',
            fontSize: '14px'
          }}
        />
        <button type="submit" style={{
          padding: '10px 20px',
          backgroundColor: '#4CAF50',
          color: 'white',
          border: 'none',
          borderRadius: '6px',
          cursor: 'pointer'
        }}>
          추가
        </button>
      </form>

      {/* 통계 */}
      <div style={{ marginBottom: '15px', fontSize: '14px', color: '#666' }}>
        전체: {totalCount}개 | 진행 중: {activeCount}개 | 완료: {completedCount}개
      </div>

      {/* 필터 버튼 */}
      <div style={{ display: 'flex', gap: '10px', marginBottom: '20px' }}>
        <button onClick={() => setFilter('all')} 
                style={getFilterButtonStyle(filter === 'all')}>
          전체
        </button>
        <button onClick={() => setFilter('active')}
                style={getFilterButtonStyle(filter === 'active')}>
          진행 중
        </button>
        <button onClick={() => setFilter('completed')}
                style={getFilterButtonStyle(filter === 'completed')}>
          완료
        </button>
      </div>

      {/* 할 일 목록 */}
      {filteredTodos.length === 0 ? (
        <p style={{ textAlign: 'center', color: '#999' }}>
          {filter === 'all' ? '할 일이 없습니다.' : 
           filter === 'active' ? '진행 중인 일이 없습니다.' : 
           '완료된 일이 없습니다.'}
        </p>
      ) : (
        <ul style={{ listStyle: 'none', padding: 0 }}>
          {filteredTodos.map(todo => (
            <li key={todo.id} style={{
              display: 'flex',
              alignItems: 'center',
              gap: '10px',
              padding: '10px',
              backgroundColor: 'white',
              marginBottom: '8px',
              borderRadius: '6px',
              boxShadow: '0 1px 3px rgba(0,0,0,0.05)'
            }}>
              <input
                type="checkbox"
                checked={todo.completed}
                onChange={() => toggleTodo(todo.id)}
              />
              <span style={{
                flex: 1,
                textDecoration: todo.completed ? 'line-through' : 'none',
                color: todo.completed ? '#999' : '#333'
              }}>
                {todo.text}
              </span>
              <button
                onClick={() => deleteTodo(todo.id)}
                style={{
                  padding: '5px 10px',
                  backgroundColor: '#f44336',
                  color: 'white',
                  border: 'none',
                  borderRadius: '4px',
                  cursor: 'pointer'
                }}
              >
                삭제
              </button>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}

function getFilterButtonStyle(isActive) {
  return {
    padding: '6px 14px',
    backgroundColor: isActive ? '#2196F3' : '#e0e0e0',
    color: isActive ? 'white' : '#333',
    border: 'none',
    borderRadius: '6px',
    cursor: 'pointer'
  };
}

export default TodoApp;
```

### 🎓 이 예제에서 배운 것들

1. ✅ **useState**로 여러 종류의 state 관리 (배열, 문자열)
2. ✅ **이벤트 핸들링** (submit, change, click)
3. ✅ **조건부 렌더링** (목록이 비어있는 경우)
4. ✅ **리스트 렌더링** (`map`과 `key`)
5. ✅ **불변성 유지** (스프레드 연산자, `filter`, `map`)
6. ✅ **컴포넌트 분리와 헬퍼 함수**

---

## 12. React 활용 방안과 다음 단계

### 🌟 React로 무엇을 만들 수 있나?

#### 1) **싱글 페이지 애플리케이션 (SPA)**
- 페이지 새로고침 없이 매끄럽게 동작하는 웹 앱
- 예: Gmail, Trello, 토스 웹

#### 2) **대시보드 / 관리자 페이지**
- 데이터 시각화, 차트, 표가 많은 화면

#### 3) **모바일 앱 (React Native)**
- 같은 React 지식으로 iOS/Android 앱 개발

#### 4) **데스크톱 앱 (Electron + React)**
- Windows, Mac, Linux용 데스크톱 프로그램

### 📚 다음에 배우면 좋은 것들

#### 🥇 우선 배워야 할 것

| 주제 | 설명 |
|------|------|
| **React Router** | 페이지 전환과 URL 관리 |
| **상태 관리 라이브러리** | Zustand, Redux Toolkit, Jotai 등 |
| **TypeScript** | 타입 안정성을 위한 JavaScript 확장 |
| **CSS-in-JS / Tailwind CSS** | 스타일링 도구 |
| **Tanstack Query (React Query)** | 서버 데이터 관리 |

#### 🥈 그 다음 배울 것

| 주제 | 설명 |
|------|------|
| **Next.js** | React 기반 풀스택 프레임워크 (가장 인기) |
| **Vite + 라이브러리 구축** | 직접 프로젝트 설정 |
| **테스팅** | Vitest, React Testing Library |
| **Storybook** | 컴포넌트 문서화 도구 |

#### 🥉 심화 학습

- **React Server Components** 깊이 이해
- **성능 최적화** (`useMemo`, `useCallback`, `React.memo`)
- **Custom Hook** 만들기
- **디자인 시스템** 구축

### 🎯 실력을 키우는 방법

#### 1) **작은 프로젝트 많이 만들기**
- 계산기, 날씨 앱, 메모장, 가계부, 영화 검색 앱 등
- 한 개의 큰 프로젝트보다 여러 개의 작은 프로젝트가 효과적

#### 2) **공식 문서 읽기**
- 한국어 공식 문서: [https://ko.react.dev/](https://ko.react.dev/)
- React 팀이 직접 작성한 가장 정확한 자료

#### 3) **다른 사람의 코드 읽기**
- GitHub에서 인기 React 프로젝트 둘러보기
- 오픈소스에 작은 기여 시도해보기

#### 4) **클론 코딩**
- 좋아하는 사이트를 따라 만들어보기
- 예: 인스타그램, 유튜브, 노션 클론

### 💼 LLMOps/MLOps/Cloud 엔지니어를 위한 React 활용

> 💡 백엔드/인프라 엔지니어도 React를 알면 매우 유용합니다.

- **ML 모델 모니터링 대시보드** 직접 구축
- **데이터 파이프라인 시각화 도구** 제작
- **내부 관리 도구 (Internal Tools)** 빠르게 개발
- **AI 모델 결과를 시각화하는 데모 페이지** 제작
- **프롬프트 엔지니어링 플레이그라운드** 구축

추천 조합:
- **React + FastAPI/Flask**: ML 모델 서빙 UI
- **React + Streamlit 비교**: 더 커스터마이징 가능
- **Next.js + Vercel**: AI 데모 빠른 배포

### 🏆 성장 로드맵 (3개월 계획)

```
1개월차: 기초 다지기
├─ 1주차: JSX, 컴포넌트, Props
├─ 2주차: State, Event, 조건부/리스트 렌더링
├─ 3주차: useEffect, API 연동
└─ 4주차: 첫 프로젝트 (Todo 앱, 계산기)

2개월차: 실전 도구 익히기
├─ 1주차: React Router (페이지 라우팅)
├─ 2주차: 상태 관리 (Zustand 또는 Context API)
├─ 3주차: Tanstack Query + API 연동
└─ 4주차: 두 번째 프로젝트 (블로그, 쇼핑몰)

3개월차: 한 단계 도약
├─ 1주차: TypeScript 기초
├─ 2주차: Next.js 입문
├─ 3주차: 테스팅 기초
└─ 4주차: 포트폴리오 프로젝트 완성
```

### 🌐 유용한 학습 리소스

#### 한국어 자료
- 📘 [React 공식 문서 (한국어)](https://ko.react.dev/)
- 🎥 인프런, 패스트캠퍼스의 React 강의
- 📺 YouTube: 코딩애플, 노마드코더, 드림코딩

#### 영어 자료
- 📘 [React 공식 영문 문서](https://react.dev/)
- 🎥 [Egghead.io](https://egghead.io/)
- 📺 YouTube: Web Dev Simplified, Theo - t3.gg, Jack Herrington

---

## 🎉 마치며

축하합니다! 여기까지 읽으셨다면 React의 핵심 개념을 모두 익히신 거예요. 이제 중요한 건 **직접 코드를 작성해보는 것**입니다.

처음에는 막막하고 어려울 수 있지만, 매일 조금씩 코드를 작성하다 보면 어느새 손에 익습니다. 다음 명언을 기억해주세요:

> 💬 "프로그래밍은 책으로 배우는 것이 아니라, 손가락으로 배우는 것이다."

작은 프로젝트부터 시작해서 점점 큰 프로젝트로 확장해 나가세요. 막힐 때는 공식 문서를 읽고, 그래도 모르겠으면 검색하거나 질문하세요. 모든 개발자가 그렇게 성장합니다.

**Happy Coding! 🚀**

---

### 📌 빠른 참고 (Cheat Sheet)

```jsx
// 1. 컴포넌트 기본 구조
function MyComponent({ prop1, prop2 }) {
  const [state, setState] = useState(초기값);
  
  useEffect(() => {
    // 부수 효과
    return () => { /* 정리 */ };
  }, [의존성]);
  
  const handleClick = () => { /* 이벤트 처리 */ };
  
  return (
    <div>
      {/* JSX */}
    </div>
  );
}

// 2. 자주 쓰는 패턴
{condition && <Component />}                    // 조건부 렌더링
{condition ? <A /> : <B />}                     // 삼항 연산자
{array.map(item => <Item key={item.id} />)}     // 리스트 렌더링
setState(prev => prev + 1)                      // 함수형 업데이트
setState({ ...obj, key: value })                // 객체 업데이트
setState([...arr, newItem])                     // 배열에 추가
setState(arr.filter(item => item.id !== id))    // 배열에서 제거
```

---

*본 강의 자료는 React 19.2 (2026년 4월 기준)을 바탕으로 작성되었습니다.*
*궁금한 점이 있다면 React 공식 문서(https://ko.react.dev)를 참고하세요!*
