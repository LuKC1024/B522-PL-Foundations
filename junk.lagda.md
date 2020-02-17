```
module junk where
```

```
open import Data.Nat
open import Data.Nat.Properties
open import Relation.Binary.PropositionalEquality using (_≡_; refl)
open Relation.Binary.PropositionalEquality.≡-Reasoning using (begin_; _≡⟨_⟩_; _∎)
open import Relation.Binary.PropositionalEquality using (sym; cong; cong₂)
```

Subtyping
---------

```
row-size : List (Id × Type) → ℕ

size : Type → ℕ
size `𝔹 = 1
size `ℕ = 1
size (A ⇒ B) = suc (size A + size B)
size (Record ρ) = suc (row-size ρ)

row-size [] = 0
row-size (⟨ x , A ⟩ ∷ ρ) = suc (size A + row-size ρ)
```

```
row-mem : Id → (A : Type) → (ρ : List (Id × Type))
   (n : ℕ) → size A + row-size ρ ≤ n → Set

sub : (A : Type) → (B : Type) → (n : ℕ) → (size A + size B ≤ n) → Set
sub `𝔹 `𝔹 (suc n) m = ⊤
sub `ℕ `ℕ (suc n) m = ⊤
sub (A ⇒ B) (C ⇒ D) (suc n) m =
  let CA = sub C A n {!!} in
  let BD = sub B D n {!!} in
  CA × BD
sub (Record ρ₁) (Record ρ₂) (suc n) m =
        (∀ x A → row-mem x A ρ₂ n {!!} → row-mem x A ρ₁ n {!!})
sub _ _ n m = ⊥

row-mem x A [] n m = ⊥
row-mem x A (⟨ y , B ⟩ ∷ ρ) 0 m = {!!}
row-mem x A (⟨ y , B ⟩ ∷ ρ) (suc n) m
    with x ≟ y
... | yes x≡y = sub A B n {!!}
... | no x≢y = row-mem x A ρ n {!!}

_<:_ : Type → Type → Set
A <: B = sub A B (size A + size B) ≤-refl

_∈_ : (Id × Type) → List (Id × Type) → Set
⟨ x , A ⟩ ∈ ρ = row-mem x A ρ (size A + row-size ρ) ≤-refl

{-
`𝔹 <: `𝔹 = ⊤
`ℕ <: `ℕ = ⊤
(A ⇒ B) <: (C ⇒ D) = C <: A  ×  B <: D
Record ρ₁ <: Record ρ₂ = 
        (∀ x A → ⟨ x , A ⟩ ∈ ρ₂ → ⟨ x , A ⟩ ∈ ρ₁)
_ <: _ = ⊥        

⟨ x , B ⟩ ∈ [] = ⊥
⟨ x , B ⟩ ∈ (⟨ y , A ⟩ ∷ ρ)
    with x ≟ y
... | yes x≡y = A <: B
... | no x≢y = ⟨ x , B ⟩ ∈ ρ
-}
```





## Canonical Forms

```
data Function : Term → Set where
  Fun-λ : ∀ {A}{N} → Function (λ: A ⇒ N)
  Fun-prim : ∀{b p k} → Function ($ (b ⇒ p) k)

canonical-fun : ∀{V A B C}
  → ∅ ⊢ V ⦂ A
  → Value V
  → A <: (B ⇒ C)
    ----------
  → Function V
canonical-fun (⊢λ ⊢V) V-λ A<:⇒ = Fun-λ
canonical-fun (⊢$ {p = base B-Nat} refl) (V-const {_} {k}) A<:⇒
    with sub-inv-fun A<:⇒
... | ⟨ A₁ , ⟨ A₂ , () ⟩ ⟩
canonical-fun (⊢$ {p = base B-Bool} refl) (V-const {_} {k}) A<:⇒
    with sub-inv-fun A<:⇒
... | ⟨ A₁ , ⟨ A₂ , () ⟩ ⟩
canonical-fun (⊢$ {p = b ⇒ p} eq) (V-const {_} {k}) A<:⇒ = Fun-prim
canonical-fun (⊢<: ⊢M A<:) V A<:⇒
    with sub-inv-fun A<:⇒
... | ⟨ A₁ , ⟨ A₂ , refl ⟩ ⟩ =
    canonical-fun ⊢M V A<: 

data Constant : Base → Term → Set where
  base-const : ∀{b k} → Constant b ($ (base b) k)

canonical-base : ∀{b V A}
  → ∅ ⊢ V ⦂ A
  → Value V
  → A <: typeof-base b
    ------------
  → Constant b V
{-
canonical-base {B-Nat} (⊢λ ⊢V) vV ()
canonical-base {B-Bool} (⊢λ ⊢V) vV ()
canonical-base {B-Nat} (⊢$ {p = base B-Nat} refl) V-const <:nat = base-const
canonical-base {B-Bool} (⊢$ {p = base B-Bool} refl) V-const <:bool = base-const
canonical-base {B-Nat} ⊢empty V-〈〉 ()
canonical-base {B-Bool} ⊢empty V-〈〉 ()
canonical-base {B-Nat} (⊢insert ⊢V ⊢V₁) (V-:= vV vV₁) ()
canonical-base {B-Bool} (⊢insert ⊢V ⊢V₁) (V-:= vV vV₁) ()
canonical-base {b} (⊢<: ⊢V x) vV A<: = canonical-base ⊢V vV {!!}
-}
```

## Progress

```
data Progress (M : Term) : Set where

  step : ∀ {N}
    → M —→ N
      ----------
    → Progress M

  done :
      Value M
      ----------
    → Progress M
```

```
progress : ∀ {M A}
  → ∅ ⊢ M ⦂ A
    ----------
  → Progress M
progress (⊢` ())
progress (⊢$ _)                             = done V-const
progress (⊢λ ⊢N)                            = done V-λ
progress (⊢· {L = L}{M}{A}{B} ⊢L ⊢M)
    with progress ⊢L
... | step L—→L′                            = step (ξ (□· M) L—→L′)
... | done VL
        with progress ⊢M
...     | step M—→M′                        = step (ξ ((L ·□) VL) M—→M′)
...     | done VM = {!!}

{-
            with canonical-fun ⊢L VL {!!}
...         | Fun-λ                         = step (β-λ VM)
...         | Fun-prim {b}{p}{k}
                with ⊢L
...             | ⊢$ refl
                with canonical-base refl ⊢M VM
...             | base-const                = step δ
-}
progress (⊢μ ⊢M)                            = step β-μ
progress (⊢let {N = N} ⊢L ⊢N)
    with progress ⊢L
... | step L—→L′                            = step (ξ (let□ N) L—→L′)
... | done VL                               = step (β-let VL)
progress ⊢empty                             = done V-〈〉
progress (⊢insert {M = M}{R}{f = f} ⊢M ⊢R)
    with progress ⊢M
... | step M—→M′                            = step (ξ (f :=□, R) M—→M′)
... | done VM
        with progress ⊢R
...     | step R—→R′                        = step (ξ ((f := M ,□) VM) R—→R′)
...     | done VR                           = done (V-:= VM VR)
progress (⊢# {R = R} {f} ⊢R f∈ρ)
    with progress ⊢R
... | step R—→R′                            = step (ξ (□# f) R—→R′)
... | done VR = {!!}
{-
    with f∈ρ
... | ∈-eq {A = A}{B} A<:B = {!!}
... | ∈-neq f∈ρ' x = {!!}
-}
progress (⊢<: {A = A}{B} ⊢M A<:B) = progress ⊢M
```


```
<:-refl : ∀ A → A <: A
<:-refl `𝔹 = {!!}
<:-refl `ℕ = {!!}
<:-refl (A ⇒ B) = {!!}
<:-refl (Record ρ) = {!!}
```

```
<:-trans : ∀{A B C} → A <: B → B <: C → A <: C
<:-trans AB BC = {!!}
{-
<:-trans <:bool <:bool = <:bool
<:-trans <:nat <:nat = <:nat
<:-trans (<:fun C1A BD1) (<:fun CC1 D1D) =
    <:fun (<:-trans CC1 C1A) (<:-trans BD1 D1D)
<:-trans (<:rec R1R2) (<:rec R2R3) = <:rec {!!}
-}
```



Predicates
----------


```
data Triangular : ℕ → ℕ → Set where
  tri-zero : Triangular 0 0
  tri-add : (k n : ℕ) → Triangular k n → Triangular (suc k) (n + suc k)
```

```
dub-div2 : (k n : ℕ) → ⌊ k + k + n /2⌋ ≡ ⌊ n /2⌋ + k
dub-div2 zero n = sym (+-identityʳ ⌊ n /2⌋)
dub-div2 (suc k) n =
  let IH = dub-div2 k n in
  begin
    ⌊ suc ((k + suc k) + n) /2⌋
  ≡⟨ cong ⌊_/2⌋ (cong suc (cong₂ _+_ (+-comm k (suc k)) refl)) ⟩
    ⌊ suc ((suc k + k) + n) /2⌋
  ≡⟨ cong ⌊_/2⌋ (cong suc (+-assoc (suc k) k n)) ⟩
    ⌊ suc (suc k + (k + n)) /2⌋
  ≡⟨ cong ⌊_/2⌋ (cong suc (+-suc zero (k + (k + n)))) ⟩
    ⌊ suc (suc (k + (k + n))) /2⌋
  ≡⟨ cong ⌊_/2⌋ (cong suc (cong suc (sym (+-assoc k k n)))) ⟩
    ⌊ suc (suc ((k + k) + n)) /2⌋
  ≡⟨ refl ⟩
    suc ⌊ k + k + n /2⌋
  ≡⟨ cong suc IH ⟩
    suc (⌊ n /2⌋ + k)
  ≡⟨ cong suc (+-comm ⌊ n /2⌋ k) ⟩
    suc (k + ⌊ n /2⌋)
  ≡⟨ +-suc zero (k + ⌊ n /2⌋) ⟩
    suc k + ⌊ n /2⌋
  ≡⟨ +-comm (suc k) ⌊ n /2⌋ ⟩
    ⌊ n /2⌋ + suc k
  ∎ 
```

```
tri-sum : (k n : ℕ) → Triangular k n → n ≡ ⌊ (k * k + k) /2⌋
tri-sum zero 0 tri-zero = refl
tri-sum (suc k) .(n + suc k) (tri-add .k n t) =
  let IH = tri-sum k n t in
  begin
    n + suc k
  ≡⟨ cong₂ _+_ IH refl ⟩
    ⌊ k * k + k /2⌋ + suc k
  ≡⟨ sym (dub-div2 (suc k) (k * k + k)) ⟩
    ⌊ ((suc k) + (suc k)) + (k * k + k) /2⌋
  ≡⟨ cong ⌊_/2⌋ (+-assoc (suc k) (suc k) (k * k + k)) ⟩
    ⌊ (suc k) + ((suc k) + (k * k + k)) /2⌋
  ≡⟨ cong ⌊_/2⌋ (cong₂ _+_ refl (+-comm (suc k) (k * k + k))) ⟩
    ⌊ (suc k) + ((k * k + k) + (suc k)) /2⌋
  ≡⟨ cong ⌊_/2⌋ (sym (+-assoc (suc k) (k * k + k) (suc k))) ⟩
    ⌊ ((suc k) + (k * k + k)) + suc k /2⌋
  ≡⟨ cong ⌊_/2⌋ (cong₂ _+_ {u = suc k} (cong₂ _+_ {x = suc k}{u = k * k + k} refl (+-comm (k * k) k)) refl) ⟩
    ⌊ ((suc k) + (suc k * k)) + suc k /2⌋
  ≡⟨ cong ⌊_/2⌋ (cong₂ _+_ {u = suc k} (cong₂ _+_ {x = suc k} refl (*-comm (suc k) k)) refl) ⟩
    ⌊ ((suc k) + (k * suc k)) + suc k /2⌋
  ≡⟨ sym (cong ⌊_/2⌋ (+-suc zero (k + k * suc k + suc k))) ⟩
    ⌊ suc (k + k * suc k + suc k) /2⌋
  ∎ 
```

Relations
----------

```
data Div2 : ℕ → ℕ → Set where
  div2-zz : Div2 0 0
  div2-1z : Div2 1 0  
  div2-level : (n m : ℕ) → Div2 n m → Div2 (suc n) (suc m) → Div2 (suc (suc n)) (suc m)
  div2-up : (n m : ℕ) → Div2 n m → Div2 (suc n) m → Div2 (suc (suc n)) (suc m)
```

```
div2-0-0 : Div2 0 0
div2-0-0 = div2-zz

div2-1-0 : Div2 1 0
div2-1-0 = div2-1z

div2-2-1 : Div2 2 1
div2-2-1 = div2-up 0 0 div2-zz div2-1z

div2-3-1 : Div2 3 1
div2-3-1 = div2-level 1 zero div2-1z div2-2-1

div2-4-2 : Div2 4 2
div2-4-2 = div2-up 2 1 div2-2-1 div2-3-1
```

```
div-+ : (m n p : ℕ) → m div n → m div p → m div (n + p)
div-+ m .m .m (div-refl .m x) (div-refl .m x₁) = div-step m m (div-refl m x)
div-+ m .m .(m + n) (div-refl .m x) (div-step n .m mp) = div-step (m + n) m (div-step n m mp)
div-+ m .(m + n) p (div-step n .m mn) mp rewrite +-assoc m n p =
  let IH = div-+ m n p mn mp in 
  div-step (n + p) m IH
```
