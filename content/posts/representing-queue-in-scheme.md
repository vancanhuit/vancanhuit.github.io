+++ 
draft = false
date = 2024-07-23T21:44:54+07:00
title = "Representing queue in Scheme"
+++

[https://sarabander.github.io/sicp/html/3_002e3.xhtml#g_t3_002e3_002e2](https://sarabander.github.io/sicp/html/3_002e3.xhtml#g_t3_002e3_002e2)

```scheme
(define (make-queue)
  (let ((front-ptr '())
        (rear-ptr '()))
    (define (empty?) (null? front-ptr))
    (define (front)
      (if (empty?)
        (error "FRONT called with an empty queue: " front-ptr)
        (car front-ptr)))
    (define (insert item)
      (let ((new-pair (cons item '())))
        (cond ((empty?) 
               (set! front-ptr new-pair)
               (set! rear-ptr new-pair))
              (else
               (set-cdr! rear-ptr new-pair)
               (set! rear-ptr new-pair)))))
    (define (delete)
      (if (empty?)
        (error "DELETE called with an empty queue: " front-ptr)
        (set! front-ptr (cdr front-ptr))))
    (define (dispatch m)
      (cond ((eq? m 'front) (front))
            ((eq? m 'empty?) (empty?))
            ((eq? m 'insert) insert)
            ((eq? m 'delete) (delete))
            ((eq? m 'print) front-ptr)
            (else (error "Operation is not supported: " m))))
    dispatch))

(define q (make-queue))
(q 'empty?)
(q 'print)
((q 'insert) 'a)
((q 'insert) 'b)
((q 'insert) 'c)
(q 'print)
(q 'empty?)
(q 'front)
(q 'delete)
(q 'print)
(q 'front)
(q 'delete)
(q 'print)
(q 'delete)
(q 'print)
(q 'empty?)
```
