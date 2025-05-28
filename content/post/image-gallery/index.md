---
title: 完善的机器上架系统设计
description: 完善的机器上架系统设计
date: 2023-08-26 00:00:00+0000
image: 2.jpg
---

您想要将机器上架流程拆分为独立的任务，并在应用层管理状态。基于 Machinery v2 的架构，我为您提供一个完整的状态管理方案。

## 完善的机器上架系统设计

### 1. 工作流状态管理 (models/workflow.go)

```go
package models

import (
    "time"
    "encoding/json"
)

type WorkflowState struct {
    WorkflowID   string    `json:"workflow_id" redis:"workflow_id"`
    ServerID     string    `json:"server_id" redis:"server_id"`
    OSType       string    `json:"os_type" redis:"os_type"`
    CurrentStep  int       `json:"current_step" redis:"current_step"`
    TotalSteps   int       `json:"total_steps" redis:"total_steps"`
    Status       string    `json:"status" redis:"status"` // pending, running, completed, failed, paused
    LastError    string    `json:"last_error,omitempty" redis:"last_error"`
    CreatedAt    time.Time `json:"created_at" redis:"created_at"`
    UpdatedAt    time.Time `json:"updated_at" redis:"updated_at"`
    CompletedAt  *time.Time `json:"completed_at,omitempty" redis:"completed_at"`
    StepResults  []StepResult `json:"step_results" redis:"step_results"`
}

type StepResult struct {
    StepIndex   int       `json:"step_index"`
    StepName    string    `json:"step_name"`
    Status      string    `json:"status"` // pending, running, completed, failed
    Result      string    `json:"result,omitempty"`
    Error       string    `json:"error,omitempty"`
    StartedAt   *time.Time `json:"started_at,omitempty"`
    CompletedAt *time.Time `json:"completed_at,omitempty"`
    TaskID      string    `json:"task_id,omitempty"`
}

type DeploymentStep struct {
    Name        string
    TaskName    string
    RetryCount  int
    Description string
}

var DeploymentSteps = []DeploymentStep{
    {Name: "reinstall_os", TaskName: "reinstall_os", RetryCount: 3, Description: "重装操作系统"},
    {Name: "install_base_env", TaskName: "install_base_env", RetryCount: 2, Description: "安装基础环境"},
    {Name: "install_docker_env", TaskName: "install_docker_env", RetryCount: 2, Description: "安装Docker环境"},
    {Name: "configure_network", TaskName: "configure_network", RetryCount: 2, Description: "配置网络"},
    {Name: "register_to_inventory", TaskName: "register_to_inventory", RetryCount: 1, Description: "注册到资产管理系统"},
}
```

### 2. 状态存储服务 (services/workflow_service.go)

```go
package services

import (
    "context"
    "encoding/json"
    "fmt"
    "time"
    "github.com/go-redis/redis/v8"
    "github.com/google/uuid"
    "your-project/models"
)

type WorkflowService struct {
    redis *redis.Client
}

func NewWorkflowService(redisClient *redis.Client) *WorkflowService {
    return &WorkflowService{redis: redisClient}
}

func (s *WorkflowService) CreateWorkflow(serverID, osType string) (*models.WorkflowState, error) {
    workflowID := uuid.New().String()
    
    workflow := &models.WorkflowState{
        WorkflowID:  workflowID,
        ServerID:    serverID,
        OSType:      osType,
        CurrentStep: 0,
        TotalSteps:  len(models.DeploymentSteps),
        Status:      "pending",
        CreatedAt:   time.Now(),
        UpdatedAt:   time.Now(),
        StepResults: make([]models.StepResult, len(models.DeploymentSteps)),
    }
    
    // 初始化步骤结果
    for i, step := range models.DeploymentSteps {
        workflow.StepResults[i] = models.StepResult{
            StepIndex: i,
            StepName:  step.Name,
            Status:    "pending",
        }
    }
    
    return workflow, s.SaveWorkflow(workflow)
}

func (s *WorkflowService) SaveWorkflow(workflow *models.WorkflowState) error {
    workflow.UpdatedAt = time.Now()
    data, err := json.Marshal(workflow)
    if err != nil {
        return err
    }
    
    key := fmt.Sprintf("workflow:%s", workflow.WorkflowID)
    return s.redis.Set(context.Background(), key, data, 24*time.Hour).Err()
}

func (s *WorkflowService) GetWorkflow(workflowID string) (*models.WorkflowState, error) {
    key := fmt.Sprintf("workflow:%s", workflowID)
    data, err := s.redis.Get(context.Background(), key).Result()
    if err != nil {
        return nil, err
    }
    
    var workflow models.WorkflowState
    err = json.Unmarshal([]byte(data), &workflow)
    return &workflow, err
}

func (s *WorkflowService) UpdateStepStatus(workflowID string, stepIndex int, status, taskID string) error {
    workflow, err := s.GetWorkflow(workflowID)
    if err != nil {
        return err
    }
    
    if stepIndex >= len(workflow.StepResults) {
        return fmt.Errorf("invalid step index: %d", stepIndex)
    }
    
    now := time.Now()
    workflow.StepResults[stepIndex].Status = status
    workflow.StepResults[stepIndex].TaskID = taskID
    
    if status == "running" {
        workflow.StepResults[stepIndex].StartedAt = &now
        workflow.Status = "running"
        workflow.CurrentStep = stepIndex
    } else if status == "completed" {
        workflow.StepResults[stepIndex].CompletedAt = &now
    }
    
    return s.SaveWorkflow(workflow)
}

func (s *WorkflowService) CompleteStep(workflowID string, stepIndex int, result string) error {
    workflow, err := s.GetWorkflow(workflowID)
    if err != nil {
        return err
    }
    
    now := time.Now()
    workflow.StepResults[stepIndex].Status = "completed"
    workflow.StepResults[stepIndex].Result = result
    workflow.StepResults[stepIndex].CompletedAt = &now
    
    // 检查是否所有步骤都完成
    if stepIndex == len(workflow.StepResults)-1 {
        workflow.Status = "completed"
        workflow.CompletedAt = &now
    }
    
    return s.SaveWorkflow(workflow)
}

func (s *WorkflowService) FailStep(workflowID string, stepIndex int, errorMsg string) error {
    workflow, err := s.GetWorkflow(workflowID)
    if err != nil {
        return err
    }
    
    workflow.StepResults[stepIndex].Status = "failed"
    workflow.StepResults[stepIndex].Error = errorMsg
    workflow.Status = "failed"
    workflow.LastError = errorMsg
    
    return s.SaveWorkflow(workflow)
}
```

### 3. 工作流执行器 (services/workflow_executor.go)

```go
package services

import (
    "context"
    "fmt"
    "time"
    "github.com/RichardKnop/machinery/v2"
    "github.com/RichardKnop/machinery/v2/tasks"
    "your-project/models"
)

type WorkflowExecutor struct {
    machineryServer *machinery.Server
    workflowService *WorkflowService
}

func NewWorkflowExecutor(server *machinery.Server, workflowService *WorkflowService) *WorkflowExecutor {
    return &WorkflowExecutor{
        machineryServer: server,
        workflowService: workflowService,
    }
}

func (e *WorkflowExecutor) StartWorkflow(workflowID string) error {
    workflow, err := e.workflowService.GetWorkflow(workflowID)
    if err != nil {
        return err
    }
    
    return e.executeNextStep(workflow)
}

func (e *WorkflowExecutor) executeNextStep(workflow *models.WorkflowState) error {
    if workflow.CurrentStep >= len(models.DeploymentSteps) {
        return fmt.Errorf("workflow already completed")
    }
    
    step := models.DeploymentSteps[workflow.CurrentStep]
    
    // 创建任务签名
    signature := &tasks.Signature{
        Name: step.TaskName,
        Args: []tasks.Arg{
            {Type: "string", Value: workflow.WorkflowID},
            {Type: "string", Value: workflow.ServerID},
            {Type: "string", Value: workflow.OSType},
            {Type: "int", Value: workflow.CurrentStep},
        },
        RetryCount: step.RetryCount,
    }
    
    // 发送任务
    asyncResult, err := e.machineryServer.SendTaskWithContext(context.Background(), signature)
    if err != nil {
        return err
    }
    
    // 更新步骤状态
    return e.workflowService.UpdateStepStatus(workflow.WorkflowID, workflow.CurrentStep, "running", asyncResult.Signature.UUID)
}

func (e *WorkflowExecutor) RetryFromStep(workflowID string, stepIndex int) error {
    workflow, err := e.workflowService.GetWorkflow(workflowID)
    if err != nil {
        return err
    }
    
    if stepIndex >= len(models.DeploymentSteps) {
        return fmt.Errorf("invalid step index: %d", stepIndex)
    }
    
    // 重置从指定步骤开始的所有步骤状态
    for i := stepIndex; i < len(workflow.StepResults); i++ {
        workflow.StepResults[i].Status = "pending"
        workflow.StepResults[i].Error = ""
        workflow.StepResults[i].Result = ""
        workflow.StepResults[i].StartedAt = nil
        workflow.StepResults[i].CompletedAt = nil
        workflow.StepResults[i].TaskID = ""
    }
    
    workflow.CurrentStep = stepIndex
    workflow.Status = "running"
    workflow.LastError = ""
    
    err = e.workflowService.SaveWorkflow(workflow)
    if err != nil {
        return err
    }
    
    return e.executeNextStep(workflow)
}

// 任务完成回调
func (e *WorkflowExecutor) OnStepCompleted(workflowID string, stepIndex int, result string) error {
    err := e.workflowService.CompleteStep(workflowID, stepIndex, result)
    if err != nil {
        return err
    }
    
    workflow, err := e.workflowService.GetWorkflow(workflowID)
    if err != nil {
        return err
    }
    
    // 如果不是最后一步，继续执行下一步
    if stepIndex < len(models.DeploymentSteps)-1 {
        workflow.CurrentStep = stepIndex + 1
        err = e.workflowService.SaveWorkflow(workflow)
        if err != nil {
            return err
        }
        return e.executeNextStep(workflow)
    }
    
    return nil
}

// 任务失败回调
func (e *WorkflowExecutor) OnStepFailed(workflowID string, stepIndex int, errorMsg string) error {
    return e.workflowService.FailStep(workflowID, stepIndex, errorMsg)
}
```

### 4. 更新的任务定义 (tasks/server_tasks.go)

```go
package tasks

import (
    "fmt"
    "time"
    "errors"
    "your-project/services"
)

var workflowExecutor *services.WorkflowExecutor

func SetWorkflowExecutor(executor *services.WorkflowExecutor) {
    workflowExecutor = executor
}

// ReinstallOS 重装系统任务
func ReinstallOS(workflowID, serverID, osType string, stepIndex int) (string, error) {
    fmt.Printf("开始重装系统 - 工作流ID: %s, 服务器ID: %s, 系统类型: %s\n", workflowID, serverID, osType)
    
    // 模拟重装系统耗时
    time.Sleep(10 * time.Second)
    
    // 模拟可能的失败情况
    if serverID == "error-server" {
        err := errors.New("重装系统失败：硬件故障")
        if workflowExecutor != nil {
            workflowExecutor.OnStepFailed(workflowID, stepIndex, err.Error())
        }
        return "", err
    }
    
    result := fmt.Sprintf("服务器 %s 系统重装完成，系统类型: %s", serverID, osType)
    fmt.Println(result)
    
    // 通知工作流执行器任务完成
    if workflowExecutor != nil {
        workflowExecutor.OnStepCompleted(workflowID, stepIndex, result)
    }
    
    return result, nil
}

// InstallBaseEnv 安装基础环境
func InstallBaseEnv(workflowID, serverID, osType string, stepIndex int) (string, error) {
    fmt.Printf("开始安装基础环境 - 工作流ID: %s, 服务器ID: %s\n", workflowID, serverID)
    
    time.Sleep(5 * time.Second)
    
    result := fmt.Sprintf("服务器 %s 基础环境安装完成", serverID)
    fmt.Println(result)
    
    if workflowExecutor != nil {
        workflowExecutor.OnStepCompleted(workflowID, stepIndex, result)
    }
    
    return result, nil
}

// InstallDockerEnv 安装Docker环境
func InstallDockerEnv(workflowID, serverID, osType string, stepIndex int) (string, error) {
    fmt.Printf("开始安装Docker环境 - 工作流ID: %s, 服务器ID: %s\n", workflowID, serverID)
    
    time.Sleep(3 * time.Second)
    
    result := fmt.Sprintf("服务器 %s Docker环境安装完成", serverID)
    fmt.Println(result)
    
    if workflowExecutor != nil {
        workflowExecutor.OnStepCompleted(workflowID, stepIndex, result)
    }
    
    return result, nil
}

// ConfigureNetwork 配置网络
func ConfigureNetwork(workflowID, serverID, osType string, stepIndex int) (string, error) {
    fmt.Printf("开始配置网络 - 工作流ID: %s, 服务器ID: %s\n", workflowID, serverID)
    
    time.Sleep(2 * time.Second)
    
    result := fmt.Sprintf("服务器 %s 网络配置完成", serverID)
    fmt.Println(result)
    
    if workflowExecutor != nil {
        workflowExecutor.OnStepCompleted(workflowID, stepIndex, result)
    }
    
    return result, nil
}

// RegisterToInventory 注册到资产管理系统
func RegisterToInventory(workflowID, serverID, osType string, stepIndex int) (string, error) {
    fmt.Printf("开始注册到资产管理系统 - 工作流ID: %s, 服务器ID: %s\n", workflowID, serverID)
    
    time.Sleep(1 * time.Second)
    
    result := fmt.Sprintf("服务器 %s 已成功注册到资产管理系统", serverID)
    fmt.Println(result)
    
    if workflowExecutor != nil {
        workflowExecutor.OnStepCompleted(workflowID, stepIndex, result)
    }
    
    return result, nil
}
```

### 5. 完整的 Gin 处理器 (handlers/server_handlers.go)

```go
package handlers

import (
    "context"
    "net/http"
    "strconv"

    "github.com/gin-gonic/gin"
    "your-project/services"
    "your-project/models"
)

type ServerHandler struct {
    workflowService  *services.WorkflowService
    workflowExecutor *services.WorkflowExecutor
}

func NewServerHandler(workflowService *services.WorkflowService, workflowExecutor *services.WorkflowExecutor) *ServerHandler {
    return &ServerHandler{
        workflowService:  workflowService,
        workflowExecutor: workflowExecutor,
    }
}

// StartServerDeployment 开始机器上架流程
func (h *ServerHandler) StartServerDeployment(c *gin.Context) {
    var req struct {
        ServerID string `json:"server_id" binding:"required"`
        OSType   string `json:"os_type" binding:"required"`
    }

    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // 创建工作流
    workflow, err := h.workflowService.CreateWorkflow(req.ServerID, req.OSType)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "创建工作流失败: " + err.Error()})
        return
    }

    // 启动工作流执行
    if err := h.workflowExecutor.StartWorkflow(workflow.WorkflowID); err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "启动工作流失败: " + err.Error()})
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "message":     "机器上架流程已启动",
        "workflow_id": workflow.WorkflowID,
        "server_id":   req.ServerID,
        "status":      "started",
        "steps":       len(models.DeploymentSteps),
    })
}

// GetWorkflowStatus 获取工作流状态
func (h *ServerHandler) GetWorkflowStatus(c *gin.Context) {
    workflowID := c.Param("id")
    
    workflow, err := h.workflowService.GetWorkflow(workflowID)
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "工作流不存在"})
        return
    }

    response := gin.H{
        "workflow_id":  workflow.WorkflowID,
        "server_id":    workflow.ServerID,
        "os_type":      workflow.OSType,
        "status":       workflow.Status,
        "current_step": workflow.CurrentStep,
        "total_steps":  workflow.TotalSteps,
        "created_at":   workflow.CreatedAt,
        "updated_at":   workflow.UpdatedAt,
        "step_results": workflow.StepResults,
    }

    if workflow.Status == "completed" && workflow.CompletedAt != nil {
        response["completed_at"] = workflow.CompletedAt
    }

    if workflow.Status == "failed" {
        response["error"] = workflow.LastError
    }

    c.JSON(http.StatusOK, response)
}

// RetryFromStep 从指定步骤重试
func (h *ServerHandler) RetryFromStep(c *gin.Context) {
    workflowID := c.Param("id")
    stepStr := c.Query("step")
    
    if stepStr == "" {
        c.JSON(http.StatusBadRequest, gin.H{"error": "缺少step参数"})
        return
    }
    
    stepIndex, err := strconv.Atoi(stepStr)
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "step参数必须是数字"})
        return
    }

    if stepIndex < 0 || stepIndex >= len(models.DeploymentSteps) {
        c.JSON(http.StatusBadRequest, gin.H{"error": "step参数超出范围"})
        return
    }

    if err := h.workflowExecutor.RetryFromStep(workflowID, stepIndex); err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "重试失败: " + err.Error()})
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "message":      "工作流重试已启动",
        "workflow_id":  workflowID,
        "restart_step": stepIndex,
        "step_name":    models.DeploymentSteps[stepIndex].Name,
    })
}

// ListWorkflows 列出所有工作流
func (h *ServerHandler) ListWorkflows(c *gin.Context) {
    // 这里可以实现分页查询逻辑
    c.JSON(http.StatusOK, gin.H{
        "message": "工作流列表功能待实现",
    })
}

// GetDeploymentSteps 获取部署步骤信息
func (h *ServerHandler) GetDeploymentSteps(c *gin.Context) {
    steps := make([]gin.H, len(models.DeploymentSteps))
    for i, step := range models.DeploymentSteps {
        steps[i] = gin.H{
            "index":       i,
            "name":        step.Name,
            "task_name":   step.TaskName,
            "description": step.Description,
            "retry_count": step.RetryCount,
        }
    }

    c.JSON(http.StatusOK, gin.H{
        "total_steps": len(models.DeploymentSteps),
        "steps":       steps,
    })
}
```

### 6. 完整的主程序 (main.go)

```go
package main

import (
    "log"

    "github.com/gin-gonic/gin"
    "github.com/go-redis/redis/v8"
    "github.com/RichardKnop/machinery/v2"
    "github.com/RichardKnop/machinery/v2/config"
    redisbroker "github.com/RichardKnop/machinery/v2/brokers/redis"
    redisbackend "github.com/RichardKnop/machinery/v2/backends/redis"
    eagerlock "github.com/RichardKnop/machinery/v2/locks/eager"
    
    "your-project/handlers"
    "your-project/services"
    "your-project/tasks"
)

func setupRedis() *redis.Client {
    return redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
        DB:   0,
    })
}

func setupMachinery() (*machinery.Server, error) {
    cnf := &config.Config{
        DefaultQueue:    "server_deployment_tasks",
        ResultsExpireIn: 3600,
        Redis: &config.RedisConfig{
            MaxIdle:                3,
            IdleTimeout:            240,
            ReadTimeout:            15,
            WriteTimeout:           15,
            ConnectTimeout:         15,
            NormalTasksPollPeriod:  1000,
            DelayedTasksPollPeriod: 500,
        },
    }

    broker := redisbroker.New(cnf, "localhost:6379", "", "", 0)
    backend := redisbackend.New(cnf, "localhost:6379", "", "", 0)
    lock := eagerlock.New()
    server := machinery.NewServer(cnf, broker, backend, lock)

    // 注册机器上架相关任务
    tasksMap := map[string]interface{}{
        "reinstall_os":           tasks.ReinstallOS,
        "install_base_env":       tasks.InstallBaseEnv,
        "install_docker_env":     tasks.InstallDockerEnv,
        "configure_network":      tasks.ConfigureNetwork,
        "register_to_inventory":  tasks.RegisterToInventory,
    }

    return server, server.RegisterTasks(tasksMap)
}

func main() {
    // 设置 Redis 客户端
    redisClient := setupRedis()
    
    // 设置 Machinery 服务器
    server, err := setupMachinery()
    if err != nil {
        log.Fatal("Failed to setup machinery server:", err)
    }

    // 创建服务
    workflowService := services.NewWorkflowService(redisClient)
    workflowExecutor := services.NewWorkflowExecutor(server, workflowService)
    
    // 设置任务执行器回调
    tasks.SetWorkflowExecutor(workflowExecutor)

    // 启动Worker
    go func() {
        worker := server.NewWorker("server_deployment_worker", 5)
        if err := worker.Launch(); err != nil {
            log.Printf("Worker failed: %v", err)
        }
    }()

    // 设置Gin路由
    r := gin.Default()
    
    serverHandler := handlers.NewServerHandler(workflowService, workflowExecutor)
    
    api := r.Group("/api/v1")
    {
        // 机器上架相关接口
        api.POST("/servers/deploy", serverHandler.StartServerDeployment)
        api.GET("/workflows/:id", serverHandler.GetWorkflowStatus)
        api.POST("/workflows/:id/retry", serverHandler.RetryFromStep)
        api.GET("/workflows", serverHandler.ListWorkflows)
        api.GET("/deployment/steps", serverHandler.GetDeploymentSteps)
    }

    r.GET("/health", func(c *gin.Context) {
        c.JSON(200, gin.H{"status": "ok"})
    })

    log.Println("机器上架服务启动在 :8080")
    r.Run(":8080")
}
```

### 7. 使用示例

**启动机器上架流程：**
```bash
curl -X POST http://localhost:8080/api/v1/servers/deploy \
  -H "Content-Type: application/json" \
  -d '{"server_id": "server-001", "os_type": "Ubuntu 20.04"}'
```

**查询工作流状态：**
```bash
curl http://localhost:8080/api/v1/workflows/{workflow_id}
```

**从第3步开始重试：**
```bash
curl -X POST "http://localhost:8080/api/v1/workflows/{workflow_id}/retry?step=2"
```

**获取部署步骤信息：**
```bash
curl http://localhost:8080/api/v1/deployment/steps
```

## 核心特性

1. **独立任务管理**：每个任务都是独立的，可以单独重试
2. **状态持久化**：使用 Redis 存储工作流状态
3. **从任意步骤重试**：支持从失败的步骤开始重新执行
4. **详细状态跟踪**：记录每个步骤的执行状态、开始时间、完成时间等
5. **自动流程控制**：任务完成后自动触发下一个步骤
