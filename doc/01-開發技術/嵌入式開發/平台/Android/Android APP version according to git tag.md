你這個需求可以透過 **Git tag + `git describe`** 的方式來實作，並在 Android Studio 的 `build.gradle` 或 `versioning script` 中自動生成版本號。以下是完整思路：

---

## 1. 使用 Git tag 當版本基準

- 在專案中建立 tag，例如：
    
    ```bash
    git tag v1.0
    git tag v1.1
    git tag v1.2
    ```
    
- 每次 release 時打上新的 tag。

---

## 2. 計算目前有幾個 tag (vercount)

可以用：

```bash
git tag | wc -l
```

這會回傳目前 tag 的數量，作為 `vercount`。

在 Gradle 裡可以透過：

```gradle
def verCount = "git tag | wc -l".execute().text.trim()
```

---

## 3. 找最近的 tag (若沒有剛好符合)

這時候用 `git describe`：

```bash
git describe --tags --abbrev=0
```

這會回傳最近的 tag，例如 `v1.1`。

如果目前 commit 剛好在 tag 上，結果就是 `v1.1`；  
如果不是，則可以用：

```bash
git describe --tags
```

會得到類似：

```
v1.1-3-gabc123
```

意思是：最近的 tag 是 `v1.1`，目前 commit 距離它有 3 個 commit。

---

## 4. 自動生成版本字串

你可以在 `build.gradle` 裡這樣寫：

```gradle
def getVersionName = { ->
    def tag = "git describe --tags --abbrev=0".execute().text.trim()
    def describe = "git describe --tags".execute().text.trim()

    if (describe == tag) {
        return tag   // 剛好在 tag 上
    } else {
        return tag + "+"  // 不在 tag 上，加上 "+"
    }
}

def getVersionCode = { ->
    return "git tag | wc -l".execute().text.trim().toInteger()
}

android {
    defaultConfig {
        versionName getVersionName()
        versionCode getVersionCode()
    }
}
```

或是在 `build.gradle.kts (kotlin)`

```gradle
fun getGitDescribe(vararg args: String): String {
    val process = ProcessBuilder("git", *args)
        .redirectOutput(ProcessBuilder.Redirect.PIPE)
        .redirectError(ProcessBuilder.Redirect.PIPE)
        .start()
    process.waitFor(1, TimeUnit.SECONDS)
    return process.inputStream.bufferedReader().readText().trim()
}

fun getVersionNameValue(): String {
    return try {
        getGitDescribe("describe", "--tags", "--dirty=+")
    } catch (e: Exception) {
        // Handle case where git is not available or not a git repo
        "1.0"
    }
}

fun getVersionCodeValue(): Int {
    try {
        val output = getGitDescribe("tag")
        if (output.isEmpty()){
            return 1
        }
        return output.lines().count()
    } catch (e: Exception) {
        // Handle case where git is not available or not a git repo
        return 1
    }
}

android {
    defaultConfig {
        versionCode = getVersionCodeValue()
        versionName =  getVersionNameValue()
    }
}

```


---

## 5. 範例輸出

- 如果 commit 剛好在 `v1.1` → `versionName = v1.1`
- 如果 commit 在 `v1.1` 之後 → `versionName = v1.1+`
- `versionCode` 則是目前 tag 的數量，例如 5。
