+++ 
draft = false
date = 2024-07-23T21:44:54+07:00
title = "Representing queues in Scheme"
+++

[https://www.gnu.org/software/mit-scheme/](https://www.gnu.org/software/mit-scheme/)

[https://sarabander.github.io/sicp/html/3_002e3.xhtml#g_t3_002e3_002e2](https://sarabander.github.io/sicp/html/3_002e3.xhtml#g_t3_002e3_002e2)

Using message-passing style.

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
        (if (empty?)
          (begin 
            (set! front-ptr new-pair)
            (set! rear-ptr new-pair)
            front-ptr)
          (begin 
            (set-cdr! rear-ptr new-pair)
            (set! rear-ptr new-pair)
            front-ptr))))
    
    (define (delete)
      (if (empty?)
        (error "DELETE called with an empty queue: " front-ptr)
        (begin
          (set! front-ptr (cdr front-ptr))
          front-ptr)))

    (define (dispatch m)
      (cond ((eq? m 'front) (front))
            ((eq? m 'empty?) (empty?))
            ((eq? m 'insert) insert)
            ((eq? m 'delete) (delete))
            (else (error "Operation is not supported: " m))))

    dispatch))

(define q (make-queue))
(q 'empty?)
((q 'insert) 'a)
((q 'insert) 'b)
((q 'insert) 'c)
(q 'empty?)
(q 'front)
(q 'delete)
(q 'front)
(q 'delete)
(q 'delete)
(q 'empty?)
(q 'front)
(q 'delete)
(q 'rear)
```
