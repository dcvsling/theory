# TypeScript 
## Type Definition and Type Inference

### 簡介

近期在研究如何用透過型別推斷定義出固定長度的陣列  
本篇將嘗試解釋給大家聽

### 研究案例

先提這次的研究案例有兩個如下 (其實只有一個):
- 定義型別為 從 0 到 指定數值的陣列型別  
```typescript
export type Sequence<N extends number, R extends unknown[] = []> = 
  R['length'] extends N 
        ? R 
        : Sequence<N, [R['length'], ...R]>;
// example
const a: Sequence<5> = [0, 1, 2, 3, 4]; // typeof a is [0,1,2,3,4]
const b: Sequence<3> = [0, 1, 2]; // typeof a is [0,1,2]
// how to use
// ex: define type is 10 | 11| 12| 13| .... | 19 
type ArrayToUnion<T> = T extends any[] ? T[number] : T; // [1,2,3,4] to 1 | 2 | 3 | 4
type UnionSequence<T extends number> = ArrayToUnion<Sequence<T>>; 
// 0 - 19 union exclude 0 - 9 union should be 10 - 19 union
type Example = Exclude<UnionSequence<20>, UnionSequence<10>> 
```
- 定義一個固定長度且自訂型別的陣列與矩陣
```typescript
type Fixed<N extends number, P = number, R extends P[] = []> = 
  R['length'] extends N 
        ? R 
        : Sequence<N, P, [P, ...R]>;
// example 
const a: Fixed<3> = [0,0,0]; //typeof a is [number, number, number]
const b: Fixed<4, string> = ['','','',''] //typeof b is [string, string, string, string]
// how to use
type Matrix<X extends number, Y extends number, T = number> = Fixed<X, Fixed<Y, T>>;
const m: Matrix<2, 3> = [[0,0,0],[1,2,2]];
const m: Matrix<3, 2, string> = [['0','0'],['1','2'],['','']];
```

想跳過解說直接使用的可以從這裡取得相關語法
[typescript playground](https://www.typescriptlang.org/play?#code/C4TwDgpgBAyhCOBXCA7AxhAPAOShAHsKgCYDOUKiAtgEYQBOANFAEp6EnmIoDWKA9gHcUAbQC6UALxRxAPilQAUFFYiA5ABtUAc2AALNRIJEUZKLmUqrKgPysl16wC5YCZOizZmIgHR+WzCzqWii6BmJisgDcigD0sewAhlRgWopo-CikwFCJLnBIqBiYAKzy0iIADMwAjMwATMwAzMwALGJRUPFQoJD8AGa5UACW5FWMdY0t7emZ2VA0+W5FWE3lMtVQdVD1HV0JvRADQ6MbE4y7cQl6Qj38UIikEFfsLsQQ-cMo0Icj5DWVKAAHy2NRBNXq4KaIL8fmBWwAnEpfgBBej0RIgAAq-AAqihhplMFj1lj2CYzIkUCBxFA7FiRJRaAwJC4sZ1uiJJoxphJgPcavD6vCmvDWopfvjCSgCu5imTjJwKNQ6PR1miMdi8QSibKVsTZNElN1AQBaREPHUodhoDSId5QM1QJHcaVQUg3RAaYgLaAAqDmmouq0S8DQACi+GSqWg0kjtvtWClmT1Hkw9UqsmYyZlyzTAMNxtippLpbL5Yrlar1Zrtbr1cUocgUAAYsN8BBiDhyUqmarmAAFBR9hiBHumcgD2kVSIKSxBTQ6fSGcdmCyOKx2NiWDcuNsdrteKAD7ywgLHyIxboEaNaJQZLI5NB79udzBrBTjaqVPbxQ7HRI-hkEcmGVZlQJAsRZkfKBiBfA9MFaZhsnoL5tHWdQ1EYNQsJw7CVz-MNjhoICRBQtDkOAVDQko6jtFotCoO6G5BDuB4nibaAAFlEio9tMAADVXcgQOYABNYSwP7KAyWkED1n3N8BOYRSuzE5gSWiaD5ioGoXB4vj8HTZoMK-Rgf0YLkLguCIYgfHT6n03jUKMlodgY0JTLUSpcJ8sRLLUGpcPqQwAtwwwOiAA)


#### 已知問題

這個做法目前仍存在上限問題 
因為是依靠遞迴完成 所以 上限目前貌似為 999 個索引
如果要考量接續應用他們的話 大約是 900 個索引
也就是 如果寫 Sequence<1000> 會在開發階段就跳出 stackoverflow 的錯誤
後續遇到會再詳細解釋問題點

### 詳解

這個做法主要是利用 v3.7 所更新時  
允許使用 recursive 來定義型別  
以及 Array<T>.length 來產生數列
尤其是後者 想出這個方法的人真的是天才
在 typescript 中如果要宣告一個值的型別為 數字 1
通常就必須先行定義出該型別
這導致可能必須先進行一串 像是宣告 keyboard code 的靜態字串
而這裡不需要這樣做 
但會有一點代價 請參考已知問題

這裡的型別定義宣告為

```typescript
type Sequence<N extends number, R extends unknown[] = []> = 
  R['length'] extends N 
        ? R 
        : Sequence<N, [R['length'], ...R]>;
```

1. 除了 數列長度 N 之外 還宣告了一個陣列 R 並給予預設值空陣列  
而這個空陣列就是用來產生數列的主要功臣
2. 判斷邏輯為 如果陣列 R 的長度是 N 就回傳陣列 R 否則 回傳 ```Sequence<N, [R['length'], ...R]>```  
```[R['length'], ...R]``` 這段語法的意思把陣列 R 前面再加上一個數字做為新的泛型參數帶回 Sequence 的 R
而有趣的事情就發生了  
此時因為陣列 R 多了一個元素 導致下一個 R['length'] 比前一次多了 1
使得這段語法會在 typescript 的 編譯器中進行遞迴
使陣列 R 持續增加元素直到他的長度等於 N
這樣我們就可以透過 Sequence<N> 來取得指定長度的數列了
有了這個 就可以做很多過去不好處理的型別或判斷

### 好處

1. 在參數傳遞上 如果需要 指定個數的元素陣列時 不用在刻死一個專用的型別 來限制  
ex: 
```typescript
//old
let numb5: [number,number,number,number,number] = [1,2,3,4,5];
function matrixAdd<T extends [[number, number], [number, number]]>(left: T, right: T);
//new
let numb5: Fixed<5> = [1,2,3,4,5];
function matrixAdd<X extends number, Y extends number>(left: Matrix<X, Y>, right: Matrix<X, Y>);
```

2. 減少 runtime 繁瑣的結構判斷, 改成於 designTime 就進行約定加以限制 以提升執行效能
ex:
```typescript
//old
function runNumb5(num5: number[]) {
    if(num5.length === 5) return;
    ///....
}
//new
function runNumb5(num5: Fixed<5>) {
    // num5 length exactly equals 5 without condition
    ///....
}
```

....more
還有很多可能性都可以從這個點延伸  

### 後記

我當初想做的其實就是 ```Matrix<X, Y>```  
當然在更早之前其實就已經開始思考該怎麼做  
只是以前無法實現  
而且這件事情對於對數理 數字有興趣的人
應該都可以腦洞大開