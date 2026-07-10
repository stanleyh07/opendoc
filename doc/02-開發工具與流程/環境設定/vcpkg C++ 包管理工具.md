以下是一份完整詳細的 **vcpkg 說明文件**，以 **Windows 環境** 為主，使用 **Markdown 格式** 撰寫，包含安裝、使用方式、注意事項，以及最後的指令速查表。

---

# vcpkg 使用說明文件 (Windows)

## 1. 什麼是 vcpkg

**vcpkg** 是由 Microsoft 開發的 **C++ 包管理工具**，用來快速安裝、更新與管理第三方 C++ 函式庫。它支援 Windows、Linux、macOS，並能與 **Visual Studio**、**MSBuild**、**CMake** 整合，讓開發者不必手動下載、編譯、設定路徑。

---

## 2. 安裝 vcpkg (Windows)

### 2.1 先決條件

- **Windows 7 以上版本**
- **Git** (需安裝並設定 PATH)
- **Visual Studio 2015 Update 3 或更新版本** (建議 VS 2022，並安裝 C++ 開發工具組)

### 2.2 安裝步驟

```powershell
# 下載 vcpkg 原始碼
git clone https://github.com/microsoft/vcpkg.git
cd vcpkg

# 編譯 vcpkg 工具
.\bootstrap-vcpkg.bat
```

完成後會生成 `vcpkg.exe`。

---

## 3. 整合到 Visual Studio / MSBuild

```powershell
.\vcpkg.exe integrate install
```

這樣 Visual Studio 專案就能自動識別 vcpkg 安裝的庫，不需額外設定 Include/Lib 路徑。

---

## 4. 基本使用方式

### 4.1 搜尋套件

```powershell
.\vcpkg.exe search boost
```

### 4.2 安裝套件

```powershell
.\vcpkg.exe install boost
```

### 4.3 移除套件

```powershell
.\vcpkg.exe remove boost
```

### 4.4 查看已安裝套件

```powershell
.\vcpkg.exe list
```

### 4.5 更新套件

```powershell
.\vcpkg.exe upgrade
```

### 4.6 使用 triplet (指定平台/編譯方式)

例如安裝 x64 靜態連結版本：

```powershell
.\vcpkg.exe install boost:x64-windows-static
```

---

## 5. 注意事項

- **路徑問題**：建議將 vcpkg 放在簡短路徑 (如 `C:\dev\vcpkg`)，避免編譯系統因路徑含空格或過長出錯。
- **更新 vcpkg**：定期執行 `git pull` 以更新 vcpkg 本身，確保套件清單最新。
- **triplet 使用**：不同 triplet (如 `x86-windows`, `x64-windows`, `x64-windows-static`) 會安裝不同版本的庫，需與專案設定一致。
- **CMake 整合**：可使用 `toolchain file` 方式整合：
    
    ```powershell
    cmake .. -DCMAKE_TOOLCHAIN_FILE=C:/dev/vcpkg/scripts/buildsystems/vcpkg.cmake
    ```
    
- **編譯時間**：部分大型庫 (如 Boost, OpenCV) 編譯時間較長，需耐心等待。

---

## 6. 指令速查表 (Cheat Sheet)

|功能|指令範例|
|---|---|
|搜尋套件|`vcpkg search <package>`|
|安裝套件|`vcpkg install <package>`|
|安裝指定 triplet|`vcpkg install <package>:x64-windows-static`|
|移除套件|`vcpkg remove <package>`|
|查看已安裝套件|`vcpkg list`|
|更新所有套件|`vcpkg upgrade`|
|更新 vcpkg 本身|`git pull`|
|整合到 Visual Studio|`vcpkg integrate install`|
|移除 VS 整合|`vcpkg integrate remove`|
|使用 CMake toolchain|`cmake .. -DCMAKE_TOOLCHAIN_FILE=C:/dev/vcpkg/scripts/buildsystems/vcpkg.cmake`|
