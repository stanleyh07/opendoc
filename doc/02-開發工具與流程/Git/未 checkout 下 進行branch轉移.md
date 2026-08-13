已知有三個 branch，main、develop、testing，當前 checkout 在 main branch 上，
如何在不影響當前 workspace，以及不切換到其他 branch 條件下，
讓 develop 轉移到 testing

```
git update-ref refs/heads/develop refs/heads/testing
```

其中 **refs/heads/develop** 可以是 branch 或是 tag，以及 remote branch ...等，
凡是可以 checkout 的 refs 都可以使用