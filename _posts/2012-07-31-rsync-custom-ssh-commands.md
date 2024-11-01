---
title: "Rsync and custom SSH commands"
date: "2012-07-31"
categories: 
  - "development"
---

Rsync is a great tool but can be a pain if you have to jump through hoops to connect via ssh such as connecting via a different port.

A simple solution is to use the **\-e** flag (also knows as --rsh=COMMAND). This flag allows you manually define the ssh command to use when connecting

```
rsync -e 'ssh -p2020' -rav ./* user@server:
```

Will allow me to connect to a server with SSH listening on port 2020
