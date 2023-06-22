---
title: "人喰いと宣教師問題をRustで解く"
emoji: "🧟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust"]
published: true
---

## 人喰いと宣教師問題
「人喰いと宣教師問題」は、論理パズルの一種で、コンピューターサイエンスの分野でよく使われる問題です。
この問題の目的は、一定の制約の下で全ての人を川の向こう岸に安全に運ぶことです。

問題の設定は次の通りです：
- 川の一方の岸には3人の宣教師と3人の人喰いがいます
- あなたの目的は、全員をボートを使って川の向こう岸に運ぶことです
- ボートには最大で2人までしか乗せることができません
- どこにいても、人喰いの数が宣教師の数を上回ると、宣教師が食べられてしまいます

9年前にプログラミングを学び始めたときにも勉強として解いたのですが([解いた時の記事](https://qiita.com/lovablepleiad/items/9ba7b8f4de96654fc0dd))

今回は人数の部分を任意の整数にしたものを解いてみます。

コードはここに置いてます。[Github](https://github.com/roger-sato/missionaries-and-cannibals-problem)

色々な解き方はあるのですが、この記事では深さ優先探索とA*アルゴリズムで解いてみます。

深さ優先探索は全探索の一種で、取りうる選択肢をすべて試していきます。
A*アルゴリズムは現在地点から目的地まで評価関数として定義して、評価関数が小さい(あるいは大きい)ものを優先的に選んでいくヒューリスティック探索アルゴリズムです。
A*アルゴリズムは適切な評価関数を定義できれば効率の良いアルゴリズムですが、そうでない場合は非常に効率の悪い探索になってしまう可能性があります。

まずは、State構造体を定義します。
State構造体はある時点での人喰いと宣教師とボートの配置を示してます。

```rust
#[derive(Clone, Eq, Ord, PartialEq)]
struct State {
    cannibals_left: i64, // 左岸にいる人喰いの数
    missionaries_left: i64, // 左岸にいる宣教師の数
    boat_left: bool, // ボートが左側にあればtrue,そうでない場合はfalse
}
```

そして、StateはいくつかのTraitを実装していますが、さらにPartialOrdとHashも独自実装します。

```rust
impl PartialOrd for State {
    fn partial_cmp(&self, other: &Self) -> Option<cmp::Ordering> {
        Some(
            score(self.cannibals_left, self.missionaries_left)
                .cmp(&score(other.cannibals_left, other.missionaries_left))
                .reverse(),
        )
    }
}

impl Hash for State {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.cannibals_left.hash(state);
        self.missionaries_left.hash(state);
        self.boat_left.hash(state);
    }
}
```

PartialOrdでは、score関数に基づいて並び替えを行っていますが、このscore関数がA*アルゴリズムの評価関数となります。
今回は左岸にいる人喰いと宣教師の合計値をコストとして評価しています。
探索開始地点では最もこの合計値が高く、ゴール地点では0になり、ゴールに近づくに従ってほぼ単調的に減少するためです。
コードとしては、以下のように定義づけられています。

```rust
fn score(cannibals_left: i64, missionaries_left: i64) -> i64 {
    cannibals_left + missionaries_left
}
```

また、深さ優先探索とA*アルゴリズムの挙動を変える方法として、Stateを保存するのにどのデータ構造を選ぶかを決めています。
RustではVec型を使えば深さ優先探索に、BinaryHeap型を使えばA*アルゴリズムになるようになっています。
コードの重複をなくすために、StateQueue traitを作成して、どのデータ構造を利用するかを一般化しています。
StateQueueは、push,pop,is_emptyが定義されており、Vec<State>とBinaryHeap<State>に対してそれぞれ実装を行っています。

```rust
trait StateQueue {
    fn push(&mut self, state: State);
    fn pop(&mut self) -> Option<State>;
    fn is_empty(&self) -> bool;
}

impl StateQueue for Vec<State> {
    fn push(&mut self, state: State) {
        self.push(state);
    }
    fn pop(&mut self) -> Option<State> {
        self.pop()
    }
    fn is_empty(&self) -> bool {
        self.is_empty()
    }
}

impl StateQueue for BinaryHeap<State> {
    fn push(&mut self, state: State) {
        self.push(state);
    }
    fn pop(&mut self) -> Option<State> {
        self.pop()
    }
    fn is_empty(&self) -> bool {
        self.is_empty()
    }
}
```


次に実際に問題を解くコードを書きます。
大まかなアルゴリズムはこんな感じです。

1. 初期の状態をキューに入れる
2. キューが空でないとき、キューから状態を1つ取り出す。殻であるときは、空の解を返して終了する
3. 取り出した状態がゴールであるならば、過去の行動履歴を返して終了する
4. 取り出した状態からの遷移(ボートに誰をどれだけ乗せるか)をすべて列挙する
5. 遷移先の状態が過去に実行されていなく、人喰いが宣教者よりも多い状況がない場合はキューに詰める
5. 新しい遷移先の状態に対して行動履歴を追加する
6. 2に戻る

```rust
fn solve<T: Default + StateQueue>(
    cannibals_num: i64, // 人喰いの数
    missionaries_num: i64, // 宣教師の数
    boat_capacity: i64, // ボートに乗ることのできる数
) -> Option<Vec<BoatMovement>> {
    // 初期状態
    let state = State {
        cannibals_left: cannibals_num,
        missionaries_left: missionaries_num,
        boat_left: true,
    };

    // 移動した結果を保持
    let mut history: HashMap<State, Vec<BoatMovement>> = HashMap::new();

    // Stateキュー
    let mut queue = T::default();

    queue.push(state.clone());

    while !queue.is_empty() {
        let state = queue.pop().unwrap();
        let cannibals_left = state.cannibals_left;
        let missionaries_left = state.missionaries_left;
        let cannibals_right = cannibals_num - cannibals_left;
        let missionaries_right = missionaries_num - missionaries_left;

        if cannibals_left == 0 && missionaries_left == 0 && !state.boat_left {
            return history.get(&state).map(|value| value.clone());
        }

        let max_cannibals_on_boat = if state.boat_left {
            cmp::min(boat_capacity, cannibals_left)
        } else {
            cmp::min(boat_capacity, cannibals_right)
        };

        let max_missionaries_on_boat = |cannibals_boat: i64| {
            if state.boat_left {
                cmp::min(boat_capacity - cannibals_boat, missionaries_left)
            } else {
                cmp::min(boat_capacity - cannibals_boat, missionaries_right)
            }
        };

        for cannibals_boat in 0..=max_cannibals_on_boat {
            for missionaries_boat in 0..=max_missionaries_on_boat(cannibals_boat) {
                if cannibals_boat + missionaries_boat == 0 {
                    continue;
                }

                fn update_counts(left: i64, right: i64, boat: i64, boat_left: bool) -> (i64, i64) {
                    if boat_left {
                        (left - boat, right + boat)
                    } else {
                        (left + boat, right - boat)
                    }
                }

                let (next_cannibals_left, next_cannibals_right) = update_counts(
                    cannibals_left,
                    cannibals_right,
                    cannibals_boat,
                    state.boat_left,
                );
                let (next_missionaries_left, next_missionaries_right) = update_counts(
                    missionaries_left,
                    missionaries_right,
                    missionaries_boat,
                    state.boat_left,
                );

                if !validate_cannibal_missionary_balance(ValidateCannibalMissionaryBalanceProp {
                    cannibals_left: next_cannibals_left,
                    missionaries_left: next_missionaries_left,
                    cannibals_right: next_cannibals_right,
                    missionaries_right: next_missionaries_right,
                    cannibals_boat,
                    missionaries_boat,
                }) {
                    continue;
                }

                let next_state = State {
                    cannibals_left: next_cannibals_left,
                    missionaries_left: next_missionaries_left,
                    boat_left: !state.boat_left,
                };

                // 実行の効率化のために、すでに訪れた状態であればスキップする
                // ただし、実行履歴が多い場合は再計算を行う
                if history.contains_key(&next_state)
                    && history.get(&next_state).unwrap().len() <= history.get(&state).unwrap().len()
                {
                    continue;
                }

                let mut next_history = match history.get(&state) {
                    Some(value) => value.clone(),
                    None => Vec::new(),
                };
                next_history.push(BoatMovement {
                    cannibals_boat,
                    missionaries_boat,
                    move_right: state.boat_left,
                });
                history.insert(next_state.clone(), next_history.clone());

                queue.push(next_state);
            }
        }
    }
    return None;
}
```

最後にこんな感じ使います。

```rust
fn main() {
    let cannibals = 3;
    let missionaries = 3;
    let boat_capacity = 2;

    let result_vec = solve::<Vec<State>>(cannibals, missionaries, boat_capacity);
    let result_heap = solve::<BinaryHeap<State>>(cannibals, missionaries, boat_capacity);
    match result_vec {
        Some(history) => {
            println!("Found solution! With Vec<State>");
            println!("===========================================================");
            println!("🧟 = cannibal");
            println!("😇 = missionary");
            println!("===========================================================");
            println!();
            println!("step counts: {}", history.len());
            print_history(&history);
        }
        None => {
            println!("===========================================================");
            println!("No solution found!");
            println!("===========================================================");
        }
    }
    match result_heap {
        Some(history) => {
            println!("Found solution! With BinaryHeap<State>");
            println!("===========================================================");
            println!("🧟 = cannibal");
            println!("😇 = missionary");
            println!("===========================================================");
            println!();
            println!("step counts: {}", history.len());
            print_history(&history);
        }
        None => {
            println!("===========================================================");
            println!("No solution found!");
            println!("===========================================================");
        }
    }
}
```

print_historyは行動履歴を出力する関数で以下の通りです

```rust
fn print_history(history: &Vec<BoatMovement>) {
    for action in history.iter() {
        println!("===========================================================");
        if action.move_right {
            println!(
                "(→) move right with {} 🧟 and {} 😇",
                action.cannibals_boat, action.missionaries_boat
            );
        } else {
            println!(
                "(←) move left with {} 🧟 and {} 😇",
                action.cannibals_boat, action.missionaries_boat
            );
        }
        println!("===========================================================");
        println!();
    }
}
```

人喰いが3人、宣教師3人、ボートに乗れる数を2人で実行すると次の通りになります。

```
Found solution! With Vec<State>
===========================================================
🧟 = cannibal
😇 = missionary
===========================================================

step counts: 11
===========================================================
(→) move right with 2 🧟 and 0 😇
===========================================================

===========================================================
(←) move left with 1 🧟 and 0 😇
===========================================================

===========================================================
(→) move right with 2 🧟 and 0 😇
===========================================================

===========================================================
(←) move left with 1 🧟 and 0 😇
===========================================================

===========================================================
(→) move right with 0 🧟 and 2 😇
===========================================================

===========================================================
(←) move left with 1 🧟 and 1 😇
===========================================================

===========================================================
(→) move right with 0 🧟 and 2 😇
===========================================================

===========================================================
(←) move left with 1 🧟 and 0 😇
===========================================================

===========================================================
(→) move right with 2 🧟 and 0 😇
===========================================================

===========================================================
(←) move left with 1 🧟 and 0 😇
===========================================================

===========================================================
(→) move right with 2 🧟 and 0 😇
===========================================================

Found solution! With BinaryHeap<State>
===========================================================
🧟 = cannibal
😇 = missionary
===========================================================

step counts: 11
===========================================================
(→) move right with 2 🧟 and 0 😇
===========================================================

===========================================================
(←) move left with 1 🧟 and 0 😇
===========================================================

===========================================================
(→) move right with 2 🧟 and 0 😇
===========================================================

===========================================================
(←) move left with 1 🧟 and 0 😇
===========================================================

===========================================================
(→) move right with 0 🧟 and 2 😇
===========================================================

===========================================================
(←) move left with 1 🧟 and 1 😇
===========================================================

===========================================================
(→) move right with 0 🧟 and 2 😇
===========================================================

===========================================================
(←) move left with 1 🧟 and 0 😇
===========================================================

===========================================================
(→) move right with 2 🧟 and 0 😇
===========================================================

===========================================================
(←) move left with 0 🧟 and 1 😇
===========================================================

===========================================================
(→) move right with 1 🧟 and 1 😇
===========================================================
```

このケースだと。深さ優先探索とA*アルゴリズムのステップ数は同じ11になってます。
人喰いが50人、宣教師50人、ボートに乗れる数を10人で実行すると次の通りになります。

```
Found solution! With Vec<State>
step counts: 37

Found solution! With BinaryHeap<State>
step counts: 25
```

このケースにするとA*アルゴリズムの方がステップ数が小さくなりました。

### おわりに

Rustの学習として書きました。
もっといい書き方があれば教えてくれるとすごく喜びます。
