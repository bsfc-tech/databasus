# MySQL 备份逻辑分析报告

**作者**: yanshaodong  
**日期**: 2026-06-30  
**项目**: Databasus - 数据库备份与管理系统

---

## 变更日志

| 日期 | 变更内容 | 作者 |
|------|---------|------|
| 2026-06-30 | 初始版本 - MySQL 备份逻辑分析 | yanshaodong |

---

## 1. MySQL 备份原因

### 1.1 业务连续性需求

MySQL 备份是数据库管理中至关重要的环节，主要原因包括：

1. **灾难恢复 (Disaster Recovery)**
   - 硬件故障（磁盘损坏、服务器崩溃）
   - 软件故障（数据库 corruption、操作系统崩溃）
   - 数据中心级别的灾难（火灾、洪水、电力中断）

2. **数据保护 (Data Protection)**
   - 意外删除或更新数据
   - 应用程序 bug 导致的数据损坏
   - 人为操作失误

3. **合规要求 (Compliance Requirements)**
   - 许多行业法规要求数据备份和可恢复性
   - 审计需求需要历史数据保留

4. **数据迁移与测试 (Data Migration & Testing)**
   - 环境克隆（生产 → 测试/开发）
   - 数据库版本升级前的安全网
   - 数据分析和报表生成

### 1.2 Databasus 项目的备份策略

根据项目 README 和代码分析，Databasus 对 MySQL 采用 **逻辑备份 (Logical Backup)** 策略：

- **支持的 MySQL 版本**: 5.7 和 8.0（仅逻辑备份）
- **备份工具**: `mysqldump`
- **备份特性**:
  - 逻辑备份（SQL 语句导出）
  - 支持压缩（zstd、传统压缩）
  - 支持加密（AES-256-GCM）
  - 支持 SSL 连接
  - 支持排除特定表
  - 支持存储过程、触发器、事件

---

## 2. 关键代码分析

### 2.1 核心备份执行流程

**文件**: `backend/internal/features/backups/backups/usecases/logical/mysql/create_backup_uc.go`

#### 2.1.1 备份入口函数

```go:59:109:backend/internal/features/backups/backups/usecases/logical/mysql/create_backup_uc.go
func (uc *CreateMysqlBackupUsecase) Execute(
	ctx context.Context,
	backup *backups_core_logical.LogicalBackup,
	backupConfig *backups_config_logical.LogicalBackupConfig,
	db *databases.Database,
	storage *storages.Storage,
	backupProgressListener func(completedMBs float64),
) (*backups_core_logical.BackupMetadata, error) {
	uc.logger.Info(
		"Creating MySQL backup via mysqldump",
		"databaseId", db.ID,
		"storageId", storage.ID,
	)

	my := db.Mysql
	if my == nil {
		return nil, fmt.Errorf("mysql database configuration is required")
	}

	if my.Database == nil || *my.Database == "" {
		return nil, fmt.Errorf("database name is required for mysqldump backups")
	}

	decryptedPassword, err := uc.fieldEncryptor.Decrypt(my.Password)
	if err != nil {
		return nil, fmt.Errorf("failed to decrypt database password: %w", err)
	}

	rawSizeMB, err := my.GetRawDbSizeMb(ctx, uc.logger, uc.fieldEncryptor)
	if err != nil {
		uc.logger.Warn("failed to fetch raw db size before backup",
			"database_id", db.ID,
			"error", err)
	} else {
		backup.BackupRawDbSizeMb = rawSizeMB
	}

	args := uc.buildMysqldumpArgs(my)

	return uc.streamToStorage(
		ctx,
		backup,
		backupConfig,
		tools.GetMysqlExecutable(my.Version, tools.MysqlExecutableMysqldump),
		args,
		decryptedPassword,
		storage,
		backupProgressListener,
		my,
	)
}
```

**关键设计要点**:

1. **密码解密**: 使用 `fieldEncryptor.Decrypt()` 解密存储的数据库密码
2. **原始数据库大小获取**: 备份前获取数据库大小，用于进度跟踪和验证
3. **动态参数构建**: 根据数据库配置动态构建 `mysqldump` 参数
4. **流式传输**: 使用 `streamToStorage()` 实现备份数据的流式传输，避免大文件占用磁盘空间

#### 2.1.2 mysqldump 参数构建

```go:111:159:backend/internal/features/backups/backups/usecases/logical/mysql/create_backup_uc.go
func (uc *CreateMysqlBackupUsecase) buildMysqldumpArgs(my *mysqltypes.MysqlDatabase) []string {
	args := []string{
		"--host=" + my.Host,
		"--port=" + strconv.Itoa(my.Port),
		"--user=" + my.Username,
		"--single-transaction",
		"--routines",
		"--set-gtid-purged=OFF",
		"--quick",
		"--skip-add-locks",
		"--verbose",
	}

	// One INSERT per row caps mysqldump memory on huge tables, but bloats the
	// dump and makes restores far slower. Opting into extended inserts batches
	// rows (mysqldump's default) for fast restores at higher backup memory.
	if !my.IsUseExtendedInsert {
		args = append(args, "--skip-extended-insert")
	}

	if my.HasPrivilege("TRIGGER") {
		args = append(args, "--triggers")
	}
	if my.HasPrivilege("EVENT") {
		args = append(args, "--events")
	}

	if my.Database != nil && *my.Database != "" {
		for _, table := range my.ExcludeTables {
			args = append(args, "--ignore-table="+*my.Database+"."+table)
		}
	}

	args = append(args, uc.getNetworkCompressionArgs(my)...)

	args = append(args, "--max-allowed-packet=1G")

	if my.IsHttps {
		args = append(args, "--ssl-mode=REQUIRED")
	} else {
		args = append(args, "--ssl-mode=DISABLED")
	}

	if my.Database != nil && *my.Database != "" {
		args = append(args, *my.Database)
	}

	return args
}
```

**参数说明**:

| 参数 | 作用 | 原因 |
|------|------|------|
| `--single-transaction` | 在单个事务中执行备份 | 保证 InnoDB 表的一致性，避免锁表 |
| `--routines` | 导出存储过程和函数 | 完整备份数据库逻辑 |
| `--set-gtid-purged=OFF` | 禁用 GTID 相关语句 | 避免在新环境恢复时的 GTID 冲突 |
| `--quick` | 逐行检索数据 | 减少内存占用，适合大表 |
| `--skip-add-locks` | 不添加 LOCK TABLES 语句 | 避免恢复时的锁冲突 |
| `--verbose` | 输出详细日志 | 便于调试和监控 |
| `--ignore-table` | 排除指定表 | 支持部分备份，减少备份大小 |

#### 2.1.3 网络压缩策略

```go:161:181:backend/internal/features/backups/backups/usecases/logical/mysql/create_backup_uc.go
func (uc *CreateMysqlBackupUsecase) getNetworkCompressionArgs(
	my *mysqltypes.MysqlDatabase,
) []string {
	const zstdCompressionLevel = 5

	switch my.Version {
	case tools.MysqlVersion80, tools.MysqlVersion84, tools.MysqlVersion9:
		if my.IsZstdSupported {
			return []string{
				"--compression-algorithms=zstd",
				fmt.Sprintf("--zstd-compression-level=%d", zstdCompressionLevel),
			}
		}

		return []string{"--compress"}
	case tools.MysqlVersion57:
		return []string{"--compress"}
	default:
		return []string{"--compress"}
	}
}
```

**压缩策略分析**:

- **MySQL 8.0+**: 优先使用 `zstd` 压缩（更高效），如果不支持则回退到传统压缩
- **MySQL 5.7**: 仅支持传统 `--compress` 参数
- **压缩级别**: zstd 使用级别 5（平衡速度和压缩率）

### 2.2 流式传输与存储

#### 2.2.1 流式传输核心逻辑

```go:183:332:backend/internal/features/backups/backups/usecases/logical/mysql/create_backup_uc.go
func (uc *CreateMysqlBackupUsecase) streamToStorage(
	parentCtx context.Context,
	backup *backups_core_logical.LogicalBackup,
	backupConfig *backups_config_logical.LogicalBackupConfig,
	mysqlBin string,
	args []string,
	password string,
	storage *storages.Storage,
	backupProgressListener func(completedMBs float64),
	myConfig *mysqltypes.MysqlDatabase,
) (*backups_core_logical.BackupMetadata, error) {
	uc.logger.Info("Streaming MySQL backup to storage", "mysqlBin", mysqlBin)

	ctx, cancel := uc.createBackupContext(parentCtx)
	defer cancel(nil)

	myCnfFile, err := uc.createTempMyCnfFile(myConfig, password)
	if err != nil {
		return nil, fmt.Errorf("failed to create .my.cnf: %w", err)
	}
	defer func() { _ = os.RemoveAll(filepath.Dir(myCnfFile)) }()

	fullArgs := append([]string{"--defaults-file=" + myCnfFile}, args...)

	cmd := exec.CommandContext(ctx, mysqlBin, fullArgs...)
	uc.logger.Info("Executing MySQL backup command", "command", cmd.String())

	// ... 执行命令并流式传输到存储
}
```

**流式传输优势**:

1. **低磁盘占用**: 不需要在本地保存完整的备份文件
2. **实时传输**: 备份数据直接流式传输到目标存储（S3、本地、FTP 等）
3. **安全性**: 使用临时 `.my.cnf` 文件传递凭据，避免密码泄露

#### 2.2.2 备份加密

```go:483:514:backend/internal/features/backups/backups/usecases/logical/mysql/create_backup_uc.go
func (uc *CreateMysqlBackupUsecase) setupBackupEncryption(
	backupID uuid.UUID,
	backupConfig *backups_config_logical.LogicalBackupConfig,
	storageWriter io.WriteCloser,
) (io.Writer, *backup_encryption.EncryptionWriter, backups_core_logical.BackupMetadata, error) {
	metadata := backups_core_logical.BackupMetadata{
		BackupID: backupID,
	}

	if backupConfig.Encryption != backups_core_enums.BackupEncryptionEncrypted {
		metadata.Encryption = backups_core_enums.BackupEncryptionNone
		uc.logger.Info("Encryption disabled for backup", "backupId", backupID)
		return storageWriter, nil, metadata, nil
	}

	masterKey, err := uc.secretKeyService.GetSecretKey()
	if err != nil {
		return nil, nil, metadata, fmt.Errorf("failed to get master key: %w", err)
	}

	encSetup, err := backup_encryption.SetupEncryptionWriter(storageWriter, masterKey, backupID)
	if err != nil {
		return nil, nil, metadata, err
	}

	metadata.EncryptionSalt = &encSetup.SaltBase64
	metadata.EncryptionIV = &encSetup.NonceBase64
	metadata.Encryption = backups_core_enums.BackupEncryptionEncrypted

	uc.logger.Info("Encryption enabled for backup", "backupId", backupID)
	return encSetup.Writer, encSetup.Writer, metadata, nil
}
```

**加密特性**:

- **AES-256-GCM**: 企业级加密标准
- **零信任存储**: 即使存储后端被攻破，备份数据也无法解密
- **动态密钥**: 每个备份使用独立的 salt 和 IV

### 2.3 备份权限验证

**文件**: `backend/internal/features/databases/databases/mysql/model.go`

```go:670:668:backend/internal/features/databases/databases/mysql/model.go
// checkBackupPermissions verifies the user has sufficient privileges for mysqldump backup.
// Required: SELECT, SHOW VIEW
func checkBackupPermissions(privileges string) error {
	requiredPrivileges := []string{"SELECT", "SHOW VIEW"}

	var missingPrivileges []string
}
```

**权限设计原则**:

- **最小权限原则**: 备份仅需 `SELECT` 和 `SHOW VIEW` 权限
- **只读用户**: 默认使用只读账户进行备份，避免数据修改风险
- **云模式强制**: 云部署模式强制要求只读权限（见 ADR-0007）

---

## 3. 备份流程图

```
┌─────────────────────────────────────────────────────────────┐
│                    MySQL 备份执行流程                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │  1. 验证数据库配置                       │
        │     - 检查 Host/Port/Username/Password  │
        │     - 检查 Database 名称                 │
        └─────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │  2. 解密数据库密码                       │
        │     - 使用 FieldEncryptor 解密           │
        └─────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │  3. 获取原始数据库大小                   │
        │     - 用于进度跟踪                       │
        │     - 用于验证备份完整性                 │
        └─────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │  4. 构建 mysqldump 参数                 │
        │     - 根据版本选择压缩算法               │
        │     - 根据权限选择导出对象               │
        │     - 处理表排除逻辑                     │
        └─────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │  5. 创建临时 .my.cnf 文件               │
        │     - 安全传递数据库凭据                 │
        │     - 设置严格权限 (600)                 │
        └─────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │  6. 执行 mysqldump 命令                 │
        │     - 流式读取 stdout                   │
        │     - 捕获 stderr 用于错误诊断           │
        └─────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │  7. 数据处理管道                        │
        │     mysqldump → zstd压缩 → 加密 → 存储  │
        └─────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │  8. 写入目标存储                         │
        │     - 本地存储                           │
        │     - S3                                 │
        │     - Google Drive                       │
        │     - FTP/SFTP                           │
        │     - 等其他存储后端                     │
        └─────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │  9. 清理资源                             │
        │     - 删除临时 .my.cnf 文件             │
        │     - 关闭所有 I/O 流                   │
        └─────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │  10. 返回备份元数据                      │
        │     - 备份大小                           │
        │     - 加密信息                           │
        │     - 备份状态                           │
        └─────────────────────────────────────────┘
```

---

## 4. 关键技术决策

### 4.1 为什么选择逻辑备份而非物理备份？

根据项目 ADR (Architecture Decision Record) 和代码分析：

1. **跨版本兼容性**: 逻辑备份（SQL 文件）可以在不同 MySQL 版本间恢复
2. **灵活性**: 可以选择性恢复特定表或数据库
3. **可读性**: SQL 文件可以直接查看和编辑
4. **云环境适应**: 逻辑备份更适合云环境和容器化部署

**物理备份的限制**:
- 需要完全相同的 MySQL 版本和操作系统
- 需要直接访问数据库文件（通常需要 root 权限）
- 恢复过程更复杂

### 4.2 为什么使用 `--single-transaction`？

```go:116:116:backend/internal/features/backups/backups/usecases/logical/mysql/create_backup_uc.go
"--single-transaction",
```

**原因**:
1. **一致性快照**: 在 InnoDB 中获取事务一致性备份
2. **无锁备份**: 不阻塞写入操作（对于生产环境至关重要）
3. **仅适用于 InnoDB**: MyISAM 表仍需要锁表

### 4.3 为什么禁用 GTID 相关语句？

```go:118:118:backend/internal/features/backups/backups/usecases/logical/mysql/create_backup_uc.go
"--set-gtid-purged=OFF",
```

**原因**:
1. **避免 GTID 冲突**: 在新环境恢复时，GTID 可能与其他实例冲突
2. **灵活性**: 允许备份在任意 MySQL 实例上恢复
3. **云环境友好**: 云数据库服务（如 RDS）通常不支持 GTID

---

## 5. 错误处理与诊断

### 5.1 错误分类

**文件**: `backend/internal/features/backups/backups/usecases/logical/mysql/create_backup_uc.go`

```go:601:626:backend/internal/features/backups/backups/usecases/logical/mysql/create_backup_uc.go
func (uc *CreateMysqlBackupUsecase) buildMysqldumpErrorMessage(
	waitErr error,
	stderrOutput []byte,
	mysqlBin string,
) error {
	stderrStr := string(stderrOutput)
	errorMsg := fmt.Sprintf(
		"%s failed: %v – stderr: %s",
		filepath.Base(mysqlBin),
		waitErr,
		stderrStr,
	)

	var exitErr *exec.ExitError
	if !errors.As(waitErr, &exitErr) {
		return errors.New(errorMsg)
	}

	exitCode := exitErr.ExitCode()

	if exitCode == exitCodeGenericError || exitCode == exitCodeConnectionError {
		return uc.handleConnectionErrors(stderrStr)
	}

	return errors.New(errorMsg)
}
```

### 5.2 常见错误与解决方案

| 错误类型 | 错误信息 | 原因 | 解决方案 |
|---------|---------|------|---------|
| 访问拒绝 | `Access denied` | 用户名/密码错误 | 检查数据库连接配置 |
| 连接拒绝 | `Can't connect` | MySQL 服务未运行 | 检查 MySQL 服务状态 |
| SSL 错误 | `SSL connection error` | SSL 配置错误 | 检查 SSL 模式设置 |
| 压缩算法不支持 | `Unknown compression algorithm` | 客户端/服务器版本不匹配 | 重新检测压缩支持 |
| 数据库不存在 | `Unknown database` | 数据库名称错误 | 检查数据库名称配置 |
| 超时 | `Timeout` | 网络或数据库性能问题 | 检查网络连接和数据库负载 |

---

## 6. 性能优化策略

### 6.1 网络压缩

- **MySQL 8.0+**: 使用 `zstd` 压缩（级别 5）
- **MySQL 5.7**: 使用传统压缩
- **优势**: 减少网络传输时间，降低存储成本

### 6.2 并行处理

- **流式处理**: 备份、压缩、加密、存储并行执行
- **进度跟踪**: 实时报告备份进度（每 1MB 报告一次）

### 6.3 内存管理

```go
const (
	copyBufferSize           = 8 * 1024 * 1024  // 8MB buffer
	progressReportIntervalMB = 1.0               // 每 1MB 报告一次
)
```

- **8MB 缓冲区**: 平衡内存使用和 I/O 效率
- **Extended Insert 控制**: 允许用户禁用批量插入以减少内存占用

---

## 7. 安全考虑

### 7.1 凭据管理

1. **临时文件**: 使用临时 `.my.cnf` 文件传递密码
2. **严格权限**: 临时文件权限设置为 `600`（仅所有者可读写）
3. **自动清理**: 备份完成后立即删除临时文件

### 7.2 加密

1. **传输加密**: 支持 SSL/TLS 连接
2. **存储加密**: AES-256-GCM 加密备份文件
3. **零信任**: 即使存储后端被攻破，数据也无法解密

### 7.3 权限控制

1. **只读用户**: 默认使用只读账户进行备份
2. **权限验证**: 在备份前验证用户权限
3. **云模式强制**: 云部署强制要求只读权限

---

## 8. 总结

### 8.1 MySQL 备份的核心价值

1. **数据安全性**: 保护数据免受丢失和损坏
2. **业务连续性**: 确保快速恢复和最小停机时间
3. **合规性**: 满足法规和审计要求
4. **灵活性**: 支持多种存储后端和恢复选项

### 8.2 Databasus 的技术优势

1. **流式处理**: 低磁盘占用，适合大规模部署
2. **多云支持**: 支持多种存储后端
3. **企业级安全**: 加密、权限控制、审计日志
4. **自动化**: 定时备份、自动清理、健康检查和通知
5. **可验证**: 自动恢复验证确保备份可用性

### 8.3 关键代码文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `backend/internal/features/backups/backups/usecases/logical/mysql/create_backup_uc.go` | MySQL 备份核心实现 |
| `backend/internal/features/databases/databases/mysql/model.go` | MySQL 数据模型 and 权限验证 |
| `backend/internal/util/tools/mysql.go` | MySQL 工具函数 and 版本管理 |
| `adr/0005-why-assets-binaries-instead-of-dbs-installation.md` | 为什么使用预编译二进制文件 |
| `adr/0007-cloud-readonly-encypted-backups.md` | 云模式只读加密备份决策 |

---

**文档结束**
