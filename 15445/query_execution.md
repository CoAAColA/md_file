# Query Execution

hashjoin的left_key以及right_key来自nlj的谓词，因此需要区分谓词的左右表达式分别来自于哪个表（内表还是外表）。



seq_scan的init未正确实现：table iterator无法init



维护一个最大size为k的小根堆，
