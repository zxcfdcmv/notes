---
title: zxcfdcmv's Notes
---
# 所有笔记
> 最近七天修改
```dataview
TABLE file.mtime AS 修改时间
FROM ""
WHERE file.mtime >= date(today) - dur(7 days)
SORT file.mtime DESC
```

# 运维
```dataview
TABLE file.mtime AS 修改时间 
FROM #运维
SORT file.mtime DESC
```

## Calico
```dataview
TABLE file.mtime AS 修改时间 
FROM #CNI插件/Calico 
SORT file.mtime DESC
```

