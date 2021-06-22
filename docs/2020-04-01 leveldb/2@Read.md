
# main
    提供快照一致性，获取当前访问快照（seq），并pin住 immutable MemLayer、MemLayer、Version，防止访问过程中发生了删除；相当于建立了短时session
    
    Todo：访问过程中触发 compact的说明
