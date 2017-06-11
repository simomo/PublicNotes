# Tips about spinlock

## Spin lock in ISR
### ***Local*** interruption must be disabled
We can use spin lock in a interrupt handler, but *local interruption* must be disabled.
The reason:
 1. ISR1 hold the spin lock sl1
 2. ISR1 is interrupted by another interruption(ISR2)
 3. ISR2 need sl1 too
 4. dead lock: ISR2 waits for ISR1 releasing the sl1, ISR1 waiting for ISR2 done its job then resume

