# HooksConfig doesn't update the FMP

HooksConfig has functions like `onDeposit`, `onBorrow`, etc... The problem with these functions is that they write in the memory, but don't update the free memory pointer. This can result in very serious bugs when memory is being used after calling these functions. Consider updating the free memory pointer