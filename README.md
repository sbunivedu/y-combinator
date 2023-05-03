# Y-Combinator

Y is a function that takes a function that could be viewed as describing
a recursive or self-referential function, and returns another function that
implements that recursive function. It is useful for defining recursive
functions in languages with no explicit support for recursion.

In Scheme, recursion can be defined using `define` or `letrec`. However,
`define` relies on a global variable space (vulnerable resource) and
`letrec` is usually implemented using a side effect. It is easier to reason
about programming languages and programs that have no side effects. Therefore
it is of theoretical interest to establish the ability to write recursive
functions without the use of side effects.

Here is a way to derive the Y-Combinator.

## Naive attempts
```scheme
(let ((fact
       (lambda (n)
         (if (= n 1)
             1
             (* n (fact  (- n 1)))))))
  (fact 5))

; fact: unbound identifier in: fact

((lambda (fact)
   (fact 5))
 (lambda (n)
   (if (= n 0) 1 (* n (fact  (- n 1))))))

; fact: unbound identifier in: fact
```

## Success attempts
```scheme
; use letrec
(letrec ((fact (lambda (n)
                 (if (= n 0) 1 (* n (fact (- n 1)))))))
  (fact 5))

; => 120

; use let and set! to simulate letrec
(let ((fact #f))
  (set! fact (lambda (n)
               (if (= n 0) 1 (* n (fact (- n 1))))))
  (fact 5))

; => 120

; pass fact to itself
(let ((fact (lambda (f n)
             (if (= n 0) 1 (* n (f f (- n 1)))))))
  (fact fact 5))

; => 120

```

## Wishful thinking
```scheme
; Generating a recursive factorial function from an almost recursive factorial function.

(let ((recursive-factorial (Y (lambda (???)
                                (lambda (n)
                                  (if (= n 0) 1 (* n (??? (- n 1)))))))))
  (recursive-factorial 5))
```

## Starting point
```scheme
(let ((fact (lambda (f n)
             (if (= n 0) 1 (* n (f f (- n 1)))))))
  (fact fact 5))
```

## Curry `(lambda (f n) ...)`
<details>
<summary>Result</summary>

```scheme
(let ((fact (lambda (f)
              (lambda (n)
                (if (= n 0) 1 (* n ((f f) (- n 1))))))))
  ((fact fact) 5))
```
</details>

## Remove `(f f)` by abstracting it out of the if-form
<details>
<summary>Result</summary>

```scheme
(let ((fact (lambda (f)
              (lambda (n)
                (let ((g (lambda (h)
                           (if (= n 0) 1 (* n (h (- n 1)))))))
                  (g (f f)))))))
  ((fact fact) 5))
```
</details>

## Rename `fact` to `i` and pass `n` to `(lambda (h) ...)`
<details>
<summary>Result</summary>

```scheme
; Passing n as parameter to (lambda (h) ...) instead of relying on outer n. As a preparation to tear things appart...

(let ((i (lambda (f)
              (lambda (n)
                (let ((g (lambda (h n)
                           (if (= n 0) 1 (* n (h (- n 1)))))))
                  (g (f f) n))))))
  ((i i) 5))
```
</details>


## Curry `(lambda (h n) ...)`
<details>
<summary>Result</summary>

```scheme
(let ((i (lambda (f)
              (lambda (n)
                (let ((g (lambda (h)
                           (lambda (n)
                             (if (= n 0) 1 (* n (h (- n 1))))))))
                  ((g (f f)) n))))))
  ((i i) 5))
```
</details>

## Move `(lambda (h) ...)` out as it has no free variables
<details>
<summary>Result</summary>

```scheme
; The lambda bound to g has no free variables, move it out.

(let ((g (lambda (h)
           (lambda (n)
             (if (= n 0) 1 (* n (h (- n 1))))))))
  (let ((i (lambda (f)
            (lambda (n)
              (let () ; empty
                ((g (f f)) n))))))
    ((i i) 5)))
```
</details>

## Remove empty `let`
<details>
<summary>Result</summary>

```scheme
(let ((g (lambda (h)
           (lambda (n)
             (if (= n 0) 1 (* n (h (- n 1))))))))
  (let ((i (lambda (f)
            (lambda (n)
              ((g (f f)) n)))))
    ((i i) 5)))
```
</details>

## Pass `g` as a parameter
<details>
<summary>Result</summary>

```scheme
; g is bound to meaning at outer level, pass it via a parameter
; We wish that the (let ((i ...))) form is a function to which g is passed as parameter.

(let ((g (lambda (h)
           (lambda (n)
             (if (= n 0) 1 (* n (h (- n 1))))))))
  ((lambda (j)
    (let ((i (lambda (f)
               (lambda (n)
                 ((j (f f)) n)))))
      ((i i) 5)))
   g))
```
</details>

## Move `5` out of `(lambda (j) ...)`
<details>
<summary>Result</summary>

```scheme
; The number 5 does not belong to the self application,
; Get 5 out of (lambda (j) ...).

(let ((g (lambda (h)
           (lambda (n)
             (if (= n 0) 1 (* n (h (- n 1))))))))
  (((lambda (j)
    (let ((i (lambda (f)
               (lambda (n)
                 ((j (f f)) n)))))
      (i i)))
   g) 5))
```
</details>

## Factor out `(lambda (j) ...)` to a named function `Y`
<details>
<summary>Result</summary>

```scheme
; Factor out (lambda (j) ...) to a named function Y

(define Y (lambda (j)
            (let ((i (lambda (f)
                       (lambda (n)
                         ((j (f f)) n)))))
              (i i))))

(let ((g (lambda (h)
           (lambda (n)
             (if (= n 0) 1 (* n (h (- n 1))))))))
  ((Y g) 5))
```
</details>

## Bind `(Y g)` to `fact`
<details>
<summary>Result</summary>

```scheme
(define Y (lambda (j)
            (let ((i (lambda (f)
                       (lambda (n)
                         ((j (f f)) n)))))
              (i i))))

(let ((fact (Y (lambda (h)
                 (lambda (n)
                   (if (= n 0) 1 (* n (h (- n 1)))))))))
  (fact 5))
```
</details>

## Write `(sum n)` using the Y-Combinator
<details>
<summary>Solution</summary>

```scheme
(let ((sum (Y (lambda (h)
                 (lambda (n)
                   (if (= n 0) 0 (+ n (h (- n 1)))))))))
  (sum 5))
; => 15
```
</details>

## Write list length function using the Y-Combinator
<details>
<summary>Solution</summary>

```scheme
(let ((length (Y (lambda (h)
                 (lambda (lst)
                   (if (null? lst) 0 (+ 1 (h (cdr lst)))))))))
  (length '(a b c)))

; => 3
```
</details>

## Write `(map f lst)` function using the Y-Combinator
<details>
<summary>Solution</summary>

```scheme
(let ((length (Y (lambda (h)
                 (lambda (f lst)
                   (if (null? lst)
                       '()
                       (cons (f (car lst)) (h f (cdr lst)))))))))
  (map add1 '(1 2 3)))

; => '(2 3 4)
```
</details>


Sources:
* https://homes.cs.aau.dk/~normark/pp/rec-higher-order-fu-slide-basic-recursion-sect.html
* The Why of Y https://dreamsongs.com/Files/WhyOfY.pdf
* https://lucasfcosta.com/2018/05/20/Y-The-Most-Beautiful-Idea-in-Computer-Science.html