---
layout: post
title: Writing Interpreters in Rust
category: blog
tags: rust calculator interpreter
---

Rustでインタプリタを製作したときの設計メモ。主要部は、トークン化後に手書きの再帰降順パーサでASTを作り評価機でそれを評価するという良くある構成。Rustでの設計に特徴がある点（文字列の取り扱い、Enumの活用、MoveとBorrow、タプルによる複数値返し、など）についての解説。

2020冬シーズンの篭もりプロジェクトとして`rc`というコマンドラインで動作する関数電卓を作成した。リポジトリは[https://github.com/nkon/rc-rs](https://github.com/nkon/rc-rs)に、設計ノートは[https://github.com/nkon/rc-rs/blob/master/NOTE.md](https://github.com/nkon/rc-rs/blob/master/NOTE.md)にて公開している。この記事は整理と閲覧性向上のために、[設計ノート](https://github.com/nkon/rc-rs/blob/master/NOTE.md)からインタプリタの設計に関して抜き出してまとめたものだ。

この記事では解説のみである。解説に必要なソースコードは引用してあるが、動作するコード自体は[`rc`のリポジトリ](https://github.com/nkon/rc-rs)を参照してほしい。


## 基本的な構成

パーサの基本に沿って、作ったものは電卓。括弧を含む四則演算（整数、浮動小数点数、複素数に対応）、組込みの定数（`pi`など）、組込みの関数（`sin`など）、ユーザ定義の変数、関数に対応する。

文字列を入力⇒[トークナイザ](#トークナイザ)でトークン列に変換⇒手書きの再帰降順[パーサ](#パーサ)でASTを構築⇒[評価器](#評価器)でASTを辿って評価する。

## トークナイザ

トークナイザ（`lexer`）は文字列（`String`）を受け取り、トークン列を返す。シグネチャとしては次のようなものになる。

トークンは`enum Token`で表される。Cでパーサを作る場合は1文字演算子をASCIIコードそのまま使ったトークンとすることが多い。Rustではきちんと`enum TokenOp`で定義し直す実装例が多いと思う。そのほうが`Enum`の漏れ検知機能などが活用できる。電卓全体としては複素数対応だがトークナイザのレベルでは複素数を取り扱わない。`1+i`は1つの複素数としてトークン化されるのではなく、`1`,`+`,`i`というトークン列として、パーサで1つの浮動小数点としてまとめる。

返されるトークン列の型は`Vec<Token>`となる。

エラーが発生する可能性があるので`Result`で包む。

エラー型は、インタプリタ全体を通して`MyError`という型を作る。ライブラリが返すエラーは`MyError`に変換してエラーメッセージとともに返される。`MyError`型を簡便に構成するために`thiserror`というクレートを使う。

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum MyError {
    #[error("lexer error: {1} {0}")]
    LexerIntError(String, std::num::ParseIntError),

    #[error("lexer error: {1} {0}")]
    LexerFloatError(String, std::num::ParseFloatError),

    #[error("parser error: {0}")]
    ParseError(String),

    #[error("eval error: {0}")]
    EvalError(String),
}

#[derive(Debug, Clone, PartialEq)]
pub enum TokenOp {
    Plus,       // +
    Minus,      // -
    Mul,        // *
    Div,        // /
    Mod,        // %
    Para,       // //
    ParenLeft,  // (
    ParenRight, // )
    Caret,      // ^
    Comma,      // ,
    Equal,      // =
    None,
}

#[derive(Debug, Clone, PartialEq)]
pub enum Token {
    Num(i128),
    FNum(f64),
    Op(TokenOp),
    Ident(String),
}

pub fn lexer(s: String) -> Result<Vec<Token>, MyError>
```

トークナイザに入力された文字列（`String`）は`char`の配列に分解してインデックスで取り扱う。`String`⇒`Vec<char>`変換のイデオムとしては下のコード例のように、`chars()`でイテレータを取り出し、`collect()`で集約する。ただし、どの型のイテレータを呼び出すかを明確化するために、代入される側に型注釈`Vec<char>`が必要となる。

文字列を1文字づつ処理するときにはイテレータを使う方が今風かもしれない。しかし、先読みなどの処理のしやすさなどから、配列として取り扱うことにもメリットはある。たとえば、この電卓では除算演算子（`/`）と並列演算子（`//`）がある。`/`を1文字読んだ後に、次の文字が`/`であるかどうかの先読みが必要となる。並列演算子は電子回路での抵抗素子の並列を表現する演算子。`a//b := (a*b)/(a+b)`である。配列のインデックスで先読みする時にはレンジチェックを行う。配列の添字がレンジオーバーすると（Cのように暴走するのではなく）実行時に`panic`するが、電卓アプリとしては`panic`終了ではなく適切にエラーハンドリングしてアプリは終了しないことが求められる。

数値トークンを取得する子関数（`tok_num`）のシグネチャは`fn tok_num(chars: &[char], index: usize) -> Result<(Token, usize), MyError>`のようになる。引数は`char`の配列と現在のインデックス。返り値は得られたトークンと更新されたインデックス（次にトークン化するべき文字列の位置を指している）のタプル。エラーが発生する可能性があるので、それらを`Result`で包む。

文字列の配列は`Vec<char>`だが、これはトークン子関数では変更されない。変更されない参照として借用を子関数に渡す。そのとき、`Vec`⇒`&[]`と簡略化される。その配列のどこからトークン化するかという位置は`usize`型のインデックスとして与える。

子関数では何文字か読み取って（食べて）その結果をトークンとして返す。トークン化を継続するために、次はどこから処理をするべきかのインデックスを返さなければならない。そのような場合、Cではインデックスの可変参照を渡して子関数で更新してもらうことが一般的だ。しかし、Rustではタプル使って何個でも値を返せるので、参照を受け取って変更するよりも返り値として結果を返すことが推奨される。このスタイルはRust API Guidelineの[C-NO-OUT](https://sinkuu.github.io/api-guidelines/predictability.html#c-no-out)にも定められている。

子関数は`Result<>`を返すが、エラー型が`MyError`で統一されているので`?`でショートカット処理が可能だ。`?`は、返り値がエラーならそのエラーを返して関数をリターンする。そうでなければ`Result`を分解して値を取り出す、という演算子。子関数のエラー型と呼び出した関数が返すエラー型が等しく（または`From`トレイトを使って変換可能）でなければならない。


```rust
pub fn lexer(s: String) -> Result<Vec<Token>, MyError> {
    let mut ret = Vec::new();

    let chars: Vec<char> = s.chars().collect();
    let mut i: usize = 0;
    while i < chars.len() {
        match chars[i] {
            '0'..='9' => {
                // `Num` or `FNum` begin from '0'..='9'.
                let (tk, j) = tok_num(&chars, i)?;
                i = j;
                ret.push(tk);
            }
            '+' => {
                ret.push(Token::Op(TokenOp::Plus));
                i += 1;
            }
            // 中略
            '/' => {
                if i + 1 < chars.len() && chars[i + 1] == '/' {
                    ret.push(Token::Op(TokenOp::Para));
                    i += 2;
                } else {
                    ret.push(Token::Op(TokenOp::Div));
                    i += 1;
                }
            }
            // 中略
            'a'..='z' | 'A'..='Z' | '_' => {
                let (tk, j) = tok_ident(&chars, i);
                i = j;
                ret.push(tk);
            }
            _ => {
                i += 1;
            }
        }
    }
    Ok(ret)
}
```

子関数自体の実装は次のようになっている。

文字列を先頭から読んでいって、整数、浮動小数点数などを場合分けして文字列⇒数値に変換し、`Token`の`enum`の枝として格納する。文字列⇒数値の変換は自前ではなく標準ライブラリを用いている。エラーが発生する可能性があるので`Result`に包む。エラーメッセージとともにエラー型は`MyError`に変換している（上述）。

```rust
fn tok_num(chars: &[char], index: usize) -> Result<(Token, usize), MyError> {
    let mut i = index;
    let mut mantissa = String::new();
    let mut exponent = String::new();
    let mut has_dot = false;
    let mut has_exponent = false;
    if chars[i] == '0' {
        if (i + 1) < chars.len() {
            i += 1;
            match chars[i] {
                '0'..='9' | 'a'..='f' | 'A'..='F' | 'x' | 'X' => {
                    return tok_num_int(chars, i);
                }
                '.' => {
                    mantissa.push('0');
                    mantissa.push(chars[i]);
                    has_dot = true;
                    i += 1;
                }
                _ => {
                    return Ok((Token::Num(0), i));
                }
            }
        } else {
            return Ok((Token::Num(0), i + 1));
        }
    }
    while i < chars.len() {
        match chars[i] {
            '0'..='9' => {
                mantissa.push(chars[i]);
                i += 1;
            }
            '_' => {
                // separator
                i += 1;
            }
            '.' => {
                mantissa.push(chars[i]);
                i += 1;
                has_dot = true;
            }
            'e' | 'E' => {
                i += 1;
                has_dot = true; // no dot but move to floating mode.
                has_exponent = true;
                if i < chars.len() {
                    let (a, b) = tok_get_num(chars, i);
                    exponent = a;
                    i = b;
                    break;
                }
            }
            _ => {
                break;
            }
        }
    }
    if !has_dot {
        match mantissa.parse::<i128>() {
            Ok(int) => {
                return Ok((Token::Num(int), i));
            }
            Err(e) => {
                return Err(MyError::LexerIntError(mantissa, e));
            }
        }
    }
    if has_exponent {
        mantissa.push('e');
        mantissa.push_str(&exponent);
    }
    // Ok((Token::FNum(mantissa.parse::<f64>()?), i))
    match mantissa.parse::<f64>() {
        Ok(float) => Ok((Token::FNum(float), i)),
        Err(e) => Err(MyError::LexerFloatError(mantissa, e)),
    }
}
```

簡易的には、`thiserror`が提供する`From`トレイトを用いて自動変換することも可能だ。その場合は、`MyError`に次のような`#[from]`を適用する。そうすることに寄って、下の呼び出し側のように、`String::parse::<f64>()`が返す`std::num::ParseFloatError`が`MyError(LexerFloatError)`に変換される。

```rust
#[derive(Error, Debug)]
pub enum MyError {
    #[error(transparent)]
    LexerFloatError(#[from] std::num::ParseFloatError),
}

fn tok_num(chars: &[char], index: usize) -> Result<(Token, usize), MyError> {
    let mut mantissa = String::new();

    /// いろいろな処理

    Ok((Token::FNum(mantissa.parse::<f64>()?), i))
}
```

### テスト

Rustではテストが言語に組込みなので簡単に書ける。Unit Testは、そのファイルの末尾に書いておくことが一般的。VS Codeを使っている場合は、`#[test]`のところがクリック可能になっていて、そこをクリックするとそのテストだけが走る。便利。バグに気づくたびに、そのバグを再現するテストケースを追加していくことは非常に良いことだ。

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_lexer() {
        assert_eq!(
            lexer("1+2+3".to_owned()).unwrap(),
            [
                Token::Num(1),
                Token::Op(TokenOp::Plus),
                Token::Num(2),
                Token::Op(TokenOp::Plus),
                Token::Num(3)
            ]
        );
    }
}
```

### `to_string()`と`to_owned()`

`"1+2+3"`という`&str[]`を`String`に変換するのは`to_string()`よりも`to_owned()`を使うことが一般的。`to_string()`は汎用のオブジェクト⇒文字列変換API。元が`&str[]`である場合には`to_owned()`の方が効率的とのことだ。


## パーサ

パーサは手書きの再帰降順パーサとして実装する。シグネチャは次のとおり。

* 識別子の辞書などが格納されている`Env`を引数として与えられる（後述）。これは途中で識別子などが追加されるかもしれないので`&mut`と変更可能な借用にしなければならない。
* トークン列をトークンの配列（`&Vec<Token>`⇒`&[Token]`に簡略化される）として与えられる。トークン列はパーサでは変更されない。不変の借用として与えられる。
* 結果はASTのルートノードである`Node`を`Result`で包んで返す。エラー型は`MyError`。

そこから再帰的に呼ばれる関数（下では`assign`など）は、トークン列を指すインデックスも引数として与えられ、更新されたインデックスをタプルとして返す（トークナイザと同様）。

`enum Node`はノードのタイプに対応する枝をもし、それぞれの枝の中に必要な情報が格納される。演算子などの引数を持つタイプの`Node`はポインタ（`Box<Node>`）で子ノードを持ち、最終的にツリーを構成する。

演算子の種類（`BinOp`での`Token`）や変数名（`Var`での`Token`）は`Token`として与えられたものをそのまま使う。しかし入力トークン列は不変の借用として与えられているので、`Node`に格納する時は`clone()`しなければならない。引数で与えられた`&[Token]`がASTより先に消滅してしまった場合に参照先がなくなってしまう。よって、`Token`は`Clone`を`#[derive(Clone)]`しておく必要がある。

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum Node {
    None,
    Num(i128),
    FNum(f64),
    CNum(Complex64),
    Unary(Token, Box<Node>),            // TokenOp, Operand
    BinOp(Token, Box<Node>, Box<Node>), // TokenOp, LHS, RHS
    Var(Token),                         // Token::Ident
    Func(Token, Vec<Node>),             // Token::Ident, args...
    Command(Token, Vec<Token>, String), // Token::Ident, args..., result-holder
}

pub fn parse(env: &mut Env, tok: &[Token]) -> Result<Node, MyError> {
    let (node, i) = assign(env, &tok, 0)?;
    if i < tok.len() {
        Err(MyError::ParseError(format!("token left: {:?} {}", tok, i)))
    } else {
        Ok(node)
    }
}
```

パーサが解釈する文法は次のとおり。再帰降順パーサは、この文法のとおりに対応する関数を実装して再帰呼出しする。そうすると、人間には理解が難しい複雑な式も文法に沿ってきちんとパースされる。手品みたいで興味深い。

コンパイラ技法の書籍では`yacc`などのパーサジェネレータの解説がなされている。ただ、かなりの範囲の文法が、そういったプッシュダウン・オートマトンではなく、手書きの再帰降順パーサでも実現可能でもある。

```rust
// <assign>  ::= <var> '=' <expr>
// <expr>    ::= <mul> ( '+' <mul> | '-' <mul> )*
// <mul>     ::= <exp> ( '*' <exp> | '/' <exp>)*
// <exp>     ::= <unary> '^' <exp> | <unary>
// <unary>   ::= <primary> | '-' <primary> | '+' <primary>
// <primary> ::= <num> | '(' <expr> ')' | <var> | <func> '(' <expr>* ',' ')'
// <num>     ::= <num> | <num> <postfix>
```

たとえば、`expr`は、まず`mul`項があって、`+` か`-` が有り、その後に`mul`項が続く。それをそのまま書き下すと次のようになる。

`+`は2項演算子として定義されている。`+`や`-`が続く場合は`loop`によってこのレベルの処理が繰り返される。下の実装では、最初の2項が`lhs`に保存される。この場合、`1+2+3`は`(1+2)+3`と解釈されることに注意。

`lhs`はLeft Hand Side（左項）、`rhs`はRight Hand Side（右項）。パーサ界隈でよく使われる略語。

```rust
// <expr>    ::= <mul> ( '+' <mul> | '-' <mul> )*
fn expr(env: &mut Env, tok: &[Token], i: usize) -> Result<(Node, usize), MyError> {
    tok_check_index(tok, i)?;                                               // レンジチェック
    let (mut lhs, mut i) = mul(env, tok, i)?;                               // まず`mul`項が来る（1）
    loop {
        if tok.len() <= i {
            return Ok((lhs, i));
        }
        let tok_orig = tok[i].clone();                                      // Nodeに格納するためにclone() 上述
        match tok[i] {
            Token::Op(TokenOp::Plus) | Token::Op(TokenOp::Minus) => {       // 次に `+`か`-`が来る
                let (rhs, j) = mul(env, tok, i + 1)?;                       // その次は`mul`項（2）
                i = j;
                lhs = Node::BinOp(tok_orig, Box::new(lhs), Box::new(rhs));  // Node::BinOp()を構築する。（3）
                                                                            // 続きの`+` `mul`があるなら`loop`で連結する。
            }
            _ => {
                return Ok((lhs, i));                                        // そうでなければ`Node`を返す。
            }
        }
    }
}
```

`1+2`の場合、上で`mul()`を呼び出している行（1,2）で、項を表す`mul()`から最終的に数値を表す`num()`にまで再帰的に呼び出され、`Node::Num(1)`、`Node::Num(2)`が返される。それらが、それぞれ`lhs`、`rhs`に束縛され、`Node::BinOp()`の行（3）で`Box`に入れられて`Node`の枝となる。

```rust
fn num(env: &mut Env, tok: &[Token], i: usize) -> Result<(Node, usize), MyError> {
    tok_check_index(tok, i)?;

    // 中略。。。。
    match tok[i] {
        Token::Num(n) => {
            if has_postfix {
                if is_complex {
                    Ok((Node::CNum(Complex64::new(0.0, n as f64 * f_postfix)), i + 3))
                } else {
                    Ok((Node::FNum(n as f64 * f_postfix), i + 2))
                }
            } else if is_complex {
                Ok((Node::CNum(Complex64::new(0.0, n as f64)), i + 2))
            } else {
                Ok((Node::Num(n), i + 1))
            }
        }
        Token::FNum(n) => {
            if has_postfix {
                if is_complex {
                    Ok((Node::CNum(Complex64::new(0.0, n * f_postfix)), i + 3))
                } else {
                    Ok((Node::FNum(n * f_postfix), i + 2))
                }
            } else if is_complex {
                Ok((Node::CNum(Complex64::new(0.0, n)), i + 2))
            } else {
                Ok((Node::FNum(n), i + 1))
            }
        }
        _ => Ok((Node::None, i)),
    }
}
```

### 関数定義

`rc`は式を評価するだけの電卓ではなく、ユーザ定義関数の機能もある。ユーザ定義関数などは、式評価とは分離したコマンドとして実装されている。

パーサは、トークン列を読み込んだ時に、識別子がコマンドと等しい場合は「コマンド」「引数」「引数…」というように解釈する。

関数を定義するコマンドは`defun`という。たとえば、引数を2倍にして返す関数`mul2`を定義することを考える。

`defun mul2 _1*2`という文字列は、`[Token::Ident(defun)`,`Token::Ident(mul2)`,`Token::Ident(_1)`,`Token::Op(TokenOp::Mul)`,`Token::Num(2)]`というトークン列になってパーサに入力される。`"defun"`という`cmd`辞書に登録済の識別子を見たパーサはこれを、`Node::Command(Token::Ident(defun), [Token::Ident(mul2),Token::Ident(_1),Token::Op(TokenOp::Mul),Token::Num(2)],"")`というコマンド・ノードを構築して格納する。`defun`がコマンド名、それ以降のトークン列はそのまま関数定義として格納される。3つ目の`String`はコマンドの実行結果を保管しておく場所。パース時点では空文字列である。

`rc`の文法として、`_1`は仮引数。評価次に実引数の1番目に置き換えられる。

パーサとしては、関数定義の文字列が入力された時はコマンドノードを生成するところまで。あとは評価器が行う。

⇒ [評価機::関数の評価](#関数の評価) に続く。

## 評価器

評価機は、パーサが返したAST（`Node`）を受け取って、各ノードを再帰的に評価して最終的には数値を返す。結果の数値は整数、浮動小数点数、複素数がある。パーサの使っているEnum、整数（`Node::Num()`）、浮動小数点数（`Node::FNum()`）、複素数（`Node::CNum()`）を流用して評価結果を返す。

たとえば、`Node::BinOp`の評価だと次のとおり。


```rust
fn eval_binop(env: &mut Env, n: &Node) -> Result<Node, MyError> {
    if let Node::BinOp(tok, lhs, rhs) = n {
        if *tok == Token::Op(TokenOp::Equal) {
            return Ok(eval_assign(env, n)?);
        }
        let lhs = eval(env, lhs)?;                                              // 左辺を評価する。返り値はNode::Num()/Node::FNum()/Node::CNum()
        let rhs = eval(env, rhs)?;                                              // 右辺を評価する
        match tok {
            Token::Op(TokenOp::Plus) => {                                       // `+`演算子の場合
                if let Node::Num(nl) = lhs {
                    if let Node::Num(nr) = rhs {
                        return Ok(Node::Num(nl + nr));
                    }
                }
                if let Node::CNum(_) = lhs {
                    return Ok(Node::CNum(
                        eval_cvalue(env, &lhs)? + eval_cvalue(env, &rhs)?,      // 足し算の実行。左辺がCNumの場合、両辺ともCNumにキャスト
                    ));
                }
                if let Node::CNum(_) = rhs {
                    return Ok(Node::CNum(
                        eval_cvalue(env, &lhs)? + eval_cvalue(env, &rhs)?,
                    ));
                }
                return Ok(Node::FNum(
                    eval_fvalue(env, &lhs)? + eval_fvalue(env, &rhs)?,
                ));
            }
            Token::Op(TokenOp::Minus) => {
                // 中略
        }
    }
    Err(MyError::EvalError(format!("binary operator: {:?}", n)))
}
```


### 関数の評価

例として、[`defun mul2 _1*2`という文字列が入力された時](#関数定義)の話を続ける。

評価器では、関数定義の実行と、定義された関数の呼び出し、の2つが実行される。

まずは関数定義の実行。

パーサの出力として、関数定義は次のコマンド・ノードが返される。

```
Node::Command(Token::Ident(defun), [Token::Ident(mul2),Token::Ident(_1),Token::Op(TokenOp::Mul),Token::Num(2)],"")
```

評価器は、次の動作を行う。

1. この`Node::Command`を実行する。
2. 最初の枝が`"defun"`なので`env`の中の`cmd`辞書を`"defun"`で検索し、
3. 得られた関数ポインタ`impl_defun`に残りの引数を与えて関数ポインタを実行する。これが`defun`というコマンドの実行。
4. その結果、`env`の中の`user_func`辞書に`<K,V>`=`<"mul2", [Token::Ident(_1),Token::Op(TokenOp::Mul),Token::Num(2)]>`が登録される。

これで、関数定義の評価の結果として`env`の`user_func`中に`mul2`が登録された。

`env`は定義されている識別子などのインタプリタの実行環境を保持するオブジェクト。ユーザ定義関数は`HashMap`を用いた辞書に格納されている。`HashMap`のキーは実行時に追加されるが、`Env`のオブジェクトが破棄されるまではライフタイムが存続していないといけないので、`<'a>`というライフタイム制約が付いている。一方、`String`や`Vec`などは代入時にヒープ上に確保して（必要なら複製して、複製した方をEnvに束縛する）`Env`オブジェクトを破壊した時に同時に破壊すれば良い。

```rust
#[derive(Clone)]
pub struct Env<'a> {
    pub constant: HashMap<&'a str, Node>,
    pub variable: HashMap<String, Node>,
    pub func: HashMap<&'a str, (TypeFn, usize)>, // (function pointer, arg num: 0=variable)
    pub user_func: HashMap<String, Vec<Token>>,  // user defined function
    pub cmd: HashMap<&'a str, (TypeCmd, usize, &'a str)>, // (function pointer, arg num: 0=variable, description)
    pub debug: bool,
    pub output_radix: u8,
    pub separate_digit: usize,
    pub float_format: FloatFormat,
    pub history_path: path::PathBuf,
    pub history_max: usize,
    pub history_index: usize,
    pub history: Vec<String>,
}
```

関数を`user_func`に登録する部分のコードは次のとおり。キー、値ともに入力トークン列からは複製されてヒープ上に確保されている。

```rust
fn impl_defun(env: &mut Env, arg: &[Token]) -> String {
    if env.is_debug() {
        eprintln!("impl_defun {:?}\r", arg);
    }
    if arg.len() < 2 {
        return "defun should have at least 2 args.".to_owned();
    }
    if let Token::Ident(id) = &arg[0] {
        let mut implement = Vec::new();
        for i in arg {
            implement.push((*i).clone());
        }
        implement.remove(0);
        env.new_user_func((*id).to_owned(), &implement);
    }
    String::from("")
}
```

評価機は`env`を引数として取っているので、以後は`user_func`辞書に`mul2`が登録された状態で評価を行う。

そこに、`mul2(3)`という文字列が与えられた場合を考える。これはトークナイザによって`[Ident(mul2),TokenOp(LeftParen),Num(3),TokenOp(RightParen)]`という配列に分解される。

パーサは`env`を検索し`"mul2"`が`user_func`に登録されているので、`Node::Func(Token::Ident(mul2),[Token::Num(3)])`という関数ノードを構築する。引数の`3`は`Vec`としてぶら下がっている。

評価器は関数ノードを受け取ると`eval_func`を呼び出す。`env`を検索すると`mul2`がユーザ定義関数なので次を行う。

1. 関数定義トークン列（`[Token::Ident(_1),Token::Op(TokenOp::Mul),Token::Num(2)]`）の仮引数`_1`を第一実引数（`Num(3)`）で置き換える。
2. 他の仮引数（`_2`など）があれば同様に置き換える。
3. 置き換えた結果のトークン列（`[Token::Num(3),Token::Op(TokenOp::Mul),Token::Num(2)]`）をパース⇒評価器で評価する。
4. 得られた値をノード（`Node::Num()`など）として返す。

```rust
fn eval_func(env: &mut Env, n: &Node) -> Result<Node, MyError> {
    if env.is_debug() {
        eprintln!("eval_func {:?}\r", n);
    }
    if let Node::Func(Token::Ident(ident), param) = n {
        if let Some(func_tuple) = env.is_func(ident.as_str()) {
            let mut params: Vec<Node> = Vec::new();
            for i in param {
                let param_value = eval(env, &i)?;
                params.push(param_value);
            }
            return Ok(func_tuple.0(env, &params));                                      // システム定義関数の場合は辞書から得られた関数ポインタを実行する
        }
        if let Some(tokens) = env.is_user_func((*ident).clone()) {                      // ユーザ定義関数の場合
            let mut params: Vec<Node> = Vec::new();
            for i in param {
                let param_value = eval(env, &i)?;
                params.push(param_value);
            }
            let mut new_tokens: Vec<Token> = Vec::new();
            for t in tokens {
                match t {
                    Token::Ident(ref id) => {
                        if id == "_1" {                                                 // `_1_を実引数で置換する
                            if params.is_empty() {
                                return Err(MyError::EvalError(format!("no parameter {:?}",new_tokens)));
                            }
                            new_tokens.append(&mut node_to_token(params[0].clone()));
                        } else if id == "_2" {
                            // 中略。。。。
                    }
                }
            }
            if env.is_debug() {
                eprintln!("eval_func re-wrote tokens: {:?}\r", new_tokens);
            }
            let func_node = parse(env, &new_tokens)?;                                   // トークン列をパースしてASTを得る
            return eval(env, &func_node);                                               // ASTを評価して返す
        }
    }
    Err(MyError::EvalError(format!("unknown function: {:?}", n)))
}
```

非常に簡単な構文、メカニズムだが、ユーザ定義関数の再帰呼出しも実現可能だ。ただし、引数の個数が9個までに制限されるのが欠点。ローカル変数の実装も難しい。

## 実装上の工夫

### clippy

インタプリタは多数の関数を再帰的に呼び出す。データをMoveしていては不都合な場合が多く借用を多用する。場合によっては借用か、`*`を用いた実体型かが混乱することがある。`clippy`はそういった場合に適切なヘルプを表示してくれるので非常に助かる。

### デバッグモード

インタプリタは、トークン、ツリー、再帰呼出しなど複雑なデータ構造や呼び出しツリーが含まれている。効率良くデバッグするためにデバッグモードを実装することが多い。データ構造のダンプや関数呼び出しツリーの表示などだ。

これらはデバッグ時だけの一時的な実装ではなく、`--debug`オプションでいつでもONにできることが便利だ。なにか不審な動作やエラーが起きたら、即座にデバッグモードをONにして再実行する。そうすれば実行状態を覗けるいつものデバッグプリントが表示されるのだ。

デバッグが完了しても、ソースコードから消す必要はない。`--debug`オプションを付けなければ通常の表示で実行できる。


