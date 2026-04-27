# SKILL: libmcpp-build-test

## 描述

在WSL Docker环境中编译、增量编译和测试libmcpp项目的标准化流程。使用华为SWR镜像仓库提供的Ubuntu 24.04基础镜像，集成ccache编译缓存加速，输出标准化的JSON构建耗时报告。

## 适用范围

- 项目：libmcpp（C++17，meson+ninja构建系统）
- 环境：Windows + WSL（Ubuntu 24.04）+ Docker
- 构建工具链：meson + ninja + g++（ccache加速）
- 测试框架：gtest + meson test

## 前置条件

1. WSL已安装Docker且当前用户有docker权限（`sudo usermod -aG docker <user>` 或 `sudo chmod 666 /var/run/docker.sock`）
2. 已拉取构建镜像：
   ```bash
   wsl docker pull swr.cn-north-4.myhuaweicloud.com/openubmc/ubuntu:24.04.2_26.03
   ```
3. 代码仓库已克隆到本地（通过git clone https://gitcode.com/openUBMC/libmcpp.git）

## 执行流程

### 步骤1：环境准备与依赖安装

在Docker容器内自动安装以下依赖：
- `ccache`：编译缓存工具
- `meson` + `ninja-build`：构建系统
- `libboost-all-dev`：Boost库
- `libgtest-dev`：gtest测试框架
- `libdbus-1-dev` + `libglib2.0-dev`：DBus/GLib依赖
- `moreutils`：ts时间戳工具（可选）

### 步骤2：配置ccache

```bash
export CCACHE_DIR="/workspace/.ccache"
export CCACHE_COMPRESS=1
export CCACHE_COMPRESSLEVEL=1
export CCACHE_SLOPPINESS="time_macros,include_file_mtime,include_file_ctime,pch_defines,locale,system_headers"
export CCACHE_PCH_EXTSUM=true
export CCACHE_BASEDIR="$PROJECT_DIR"
export CCACHE_NOHASHDIR=1
ccache -M 5G
```

创建ccache编译器包装器：
```bash
mkdir -p /workspace/ccache-bin
ln -sf "$(command -v ccache)" /workspace/ccache-bin/g++
ln -sf "$(command -v ccache)" /workspace/ccache-bin/gcc
export PATH="/workspace/ccache-bin:$PATH"
```

### 步骤3：全量构建（build）

1. 清理构建目录：`rm -rf builddir`
2. 重置ccache统计：`ccache -z`
3. Meson配置：`meson setup builddir -Dtests=true -Denable_conan_compile=false`
4. 编译：`ninja -C builddir -j$(nproc)`（通常372个目标）
5. 记录耗时：meson setup耗时、ninja compile耗时、总耗时
6. 获取ccache统计：`ccache -sv`

### 步骤4：增量构建（incremental build）

1. 重置ccache统计：`ccache -z`
2. 修改至少3个源文件触发增量编译（推荐：config_manager.cpp、timer.cpp、test_timer.cpp）
   ```bash
   echo "// incremental modification $(date +%s)" >> src/core/config_manager.cpp
   echo "// incremental modification $(date +%s)" >> src/core/timer.cpp
   echo "// incremental test modification $(date +%s)" >> tests/core/test_timer.cpp
   ```
3. 增量编译：`ninja -C builddir -j$(nproc)`
4. 统计实际编译/链接文件数、耗时
5. 获取ccache统计：`ccache -sv`，计算命中率

### 步骤5：UT测试（UT）

```bash
meson test -C builddir --num-processes $(nproc) --print-errorlogs
```

从meson日志解析结果：
```bash
TEST_LOG="builddir/meson-logs/testlog.txt"
# 取第一个测试套件的结果（libmcpp主测试）
TESTS_RUN=$(grep -oP '\[==========\]\s+\K\d+(?=\s+tests)' "$TEST_LOG" | head -1)
TESTS_PASSED=$(grep -oP '\[  PASSED  \]\s+\K\d+' "$TEST_LOG" | head -1)
TESTS_SKIPPED=$(grep -oP '\[  SKIPPED \]\s+\K\d+' "$TEST_LOG" | head -1)
TESTS_FAILED=$(grep -oP '\[  FAILED  \]\s+\K\d+' "$TEST_LOG" | head -1 || \
               grep -oP '\K\d+(?=\s+FAILED TESTS)' "$TEST_LOG" | head -1)
```

### 步骤6：输出JSON报告

输出到 `build/timing_results.json`，字段命名规范：
- `build`：编译
- `incremental_build`：增量构建
- `ut`：UT测试

每个数值字段包含 `_value`（数值）和 `_comment`（中文说明）。

## 目录结构

```
<项目根目录>/
├── build/                           # 构建产物目录
│   ├── timing_results.json          # 构建耗时JSON报告
│   └── builddir/                    # meson构建目录
├── .claude/skills/
│   └── libmcpp-build-test.md        # 本SKILL文档
├── scripts/smart_build.sh           # 项目自带的智能构建脚本
└── ...
```

## Docker运行方式

```bash
wsl docker run --rm \
  -v /mnt/d/project/TTFHW/openUBMC/libmcpp/libmcpp-repo:/workspace/libmcpp:rw \
  -v /mnt/d/project/TTFHW/openUBMC/libmcpp/build:/workspace/build:rw \
  swr.cn-north-4.myhuaweicloud.com/openubmc/ubuntu:24.04.2_26.03 \
  bash -c 'cd /workspace && bash /workspace/libmcpp/../build/build_script.sh'
```

## 注意事项

1. **Docker权限问题**：WSL中需确保用户有docker访问权限，执行 `sudo usermod -aG docker wangqi` 或 `sudo chmod 666 /var/run/docker.sock`
2. **路径映射**：Windows路径通过 `/mnt/d/...` 映射到WSL，再通过 `-v` 挂载到Docker容器
3. **ccache有效性**：增量构建中未修改的文件通过ccache缓存复用，命中率应接近100%
4. **已知测试失败**：
   - `socket_appender_test`（4个）：容器内无网络服务，属预期
   - 18个skipped测试：依赖mock实现或外部服务（ShmTreeQuery、mdb_access、Match等）
5. **清理构建**：如ccache缓存异常，可删除 `build/.ccache` 目录重置
6. **使用项目自带脚本**：`scripts/smart_build.sh` 提供了更智能的构建能力（自动检测系统资源、计算最优并发数等），可在非Docker环境下直接使用

## 输出示例

```json
{
  "_comment": "libmcpp 编译与测试耗时统计",
  "build": {
    "clean_build_total_ms": { "_value": 830398, "_comment": "全量构建总耗时（毫秒）" },
    "meson_setup_ms": { "_value": 4458, "_comment": "meson setup配置耗时" },
    "ninja_compile_ms": { "_value": 825935, "_comment": "ninja compile编译耗时（372个目标）" }
  },
  "incremental_build": {
    "ninja_compile_ms": { "_value": 71093, "_comment": "增量编译耗时（3个文件重编译）" },
    "ccache": {
      "hits": { "_value": 3, "_comment": "增量构建缓存命中数" },
      "hit_rate_percent": { "_value": 100.0, "_comment": "增量构建缓存命中率" }
    }
  },
  "ut": {
    "test_total_ms": { "_value": 20310, "_comment": "UT测试总耗时" },
    "tests_run": { "_value": 1364, "_comment": "测试用例总数" },
    "tests_passed": { "_value": 1342, "_comment": "通过数" },
    "tests_failed": { "_value": 4, "_comment": "失败数（socket_appender）" }
  }
}
```
