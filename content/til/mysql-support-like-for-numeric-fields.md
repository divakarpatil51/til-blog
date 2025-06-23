+++
date = '2025-06-23T14:58:45+10:00'
title = 'MySQL Supports LIKE On Numeric Fields'
+++

While helping a friend debug a MySQL query, I learned that MySQL supports the `LIKE` condition on **numeric fields** by implicitly casting them to **strings**. However, this can cause issues: numbers like `0.xx` may lose their **leading zero** during the cast (for example `0.12` becomes `.12`), which could affect pattern matching.
