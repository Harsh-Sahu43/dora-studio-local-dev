# Dora Studio - Test Plan

> **Comprehensive Test Specifications** - Unit tests, integration tests, and E2E scenarios

## Quick Links

- [IMPLEMENTATION_GUIDE.md](IMPLEMENTATION_GUIDE.md) - Implementation roadmap
- [ISSUES.md](ISSUES.md) - Trackable issues
- [ARCHITECTURE.md](ARCHITECTURE.md) - Technical architecture

---

## Test Strategy Overview

### Test Pyramid

```
                    ┌─────────────┐
                    │    E2E      │  ← Few, slow, high confidence
                    │   Tests     │
                    ├─────────────┤
                    │ Integration │  ← Medium count, medium speed
                    │   Tests     │
                    ├─────────────┤
                    │             │
                    │    Unit     │  ← Many, fast, isolated
                    │   Tests     │
                    │             │
                    └─────────────┘
```

### Coverage Targets

| Component | Unit | Integration | E2E |
|-----------|------|-------------|-----|
| dora-studio-client | 80% | 70% | - |
| dora-studio-widgets | 60% | - | - |
| Mini-apps | 70% | 60% | 50% |
| AI Agent | 75% | 60% | - |

### Test Categories

| Category | Description | Tools |
|----------|-------------|-------|
| Unit | Isolated function/module tests | `cargo test` |
| Integration | Cross-module, mock external | `cargo test --test integration` |
| E2E | Full app with real Dora | Custom harness |
| UI | Widget rendering, interaction | Makepad test utils |
| Performance | Latency, memory, FPS | `criterion`, custom benchmarks |

---

## Phase 0: Foundation Tests

### F-01: Workspace Structure

**Unit Tests**: N/A (structural)

**Integration Tests**:
```rust
#[test]
fn test_workspace_compiles() {
    // Verify cargo check passes on workspace
    let status = Command::new("cargo")
        .args(["check", "--workspace"])
        .status()
        .expect("cargo check failed");
    assert!(status.success());
}

#[test]
fn test_all_crates_exist() {
    let crates = [
        "dora-studio-shell",
        "dora-studio-widgets",
        "dora-studio-client",
        "apps/dataflow-manager",
        "apps/yaml-editor",
        "apps/log-viewer",
        "apps/telemetry-dashboard",
    ];
    for crate_path in crates {
        assert!(Path::new(crate_path).join("Cargo.toml").exists());
    }
}
```

---

### F-06: Theme System

**Unit Tests**:
```rust
// tests/theme_test.rs

#[test]
fn test_color_constants_valid() {
    // All colors should be valid Vec4 with values 0.0-1.0
    let colors = [DARK_BG, PANEL_BG, BORDER, TEXT_PRIMARY, ACCENT_BLUE];
    for color in colors {
        assert!(color.x >= 0.0 && color.x <= 1.0, "Red channel out of range");
        assert!(color.y >= 0.0 && color.y <= 1.0, "Green channel out of range");
        assert!(color.z >= 0.0 && color.z <= 1.0, "Blue channel out of range");
        assert!(color.w >= 0.0 && color.w <= 1.0, "Alpha channel out of range");
    }
}

#[test]
fn test_contrast_ratios() {
    // TEXT_PRIMARY on DARK_BG should have sufficient contrast (WCAG AA: 4.5:1)
    let contrast = calculate_contrast_ratio(TEXT_PRIMARY, DARK_BG);
    assert!(contrast >= 4.5, "Insufficient contrast: {}", contrast);
}

#[test]
fn test_accent_colors_distinguishable() {
    // Accent colors should be visually distinct
    let accents = [ACCENT_BLUE, ACCENT_GREEN, ACCENT_RED, ACCENT_YELLOW];
    for (i, a) in accents.iter().enumerate() {
        for (j, b) in accents.iter().enumerate() {
            if i != j {
                let distance = color_distance(*a, *b);
                assert!(distance > 0.3, "Colors {} and {} too similar", i, j);
            }
        }
    }
}
```

---

### F-07: DoraApp Trait

**Unit Tests**:
```rust
// tests/app_trait_test.rs

#[test]
fn test_app_info_creation() {
    let info = AppInfo {
        name: "Test App",
        id: "test-app",
        description: "A test application",
        icon: "test-icon",
    };
    assert_eq!(info.name, "Test App");
    assert_eq!(info.id, "test-app");
}

#[test]
fn test_app_registry_register() {
    let mut registry = AppRegistry::new();
    assert_eq!(registry.len(), 0);

    registry.register(AppInfo {
        name: "App 1",
        id: "app-1",
        description: "",
        icon: "",
    });
    assert_eq!(registry.len(), 1);
}

#[test]
fn test_app_registry_find_by_id() {
    let mut registry = AppRegistry::new();
    registry.register(AppInfo {
        name: "Dataflow Manager",
        id: "dataflow-manager",
        description: "",
        icon: "",
    });

    let found = registry.find_by_id("dataflow-manager");
    assert!(found.is_some());
    assert_eq!(found.unwrap().name, "Dataflow Manager");

    let not_found = registry.find_by_id("nonexistent");
    assert!(not_found.is_none());
}

#[test]
fn test_app_registry_no_duplicates() {
    let mut registry = AppRegistry::new();
    let info = AppInfo {
        name: "App",
        id: "app",
        description: "",
        icon: "",
    };

    registry.register(info.clone());
    registry.register(info.clone()); // Should not duplicate

    // Implementation should handle duplicates
    let count = registry.apps().iter().filter(|a| a.id == "app").count();
    assert_eq!(count, 1);
}
```

---

## Phase 1: Core Services Tests

### C-02: Coordinator TCP Client

**Unit Tests**:
```rust
// dora-studio-client/src/client.rs tests

#[cfg(test)]
mod tests {
    use super::*;
    use tokio::net::TcpListener;

    #[tokio::test]
    async fn test_client_connect_success() {
        // Start mock server
        let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
        let addr = listener.local_addr().unwrap();

        // Accept connection in background
        tokio::spawn(async move {
            let _ = listener.accept().await;
        });

        let client = DoraClient::connect(&addr.to_string()).await;
        assert!(client.is_ok());
    }

    #[tokio::test]
    async fn test_client_connect_failure() {
        // Try to connect to non-existent server
        let result = DoraClient::connect("127.0.0.1:59999").await;
        assert!(result.is_err());
    }

    #[tokio::test]
    async fn test_client_reconnect_on_disconnect() {
        // TODO: Test reconnection logic
    }
}
```

**Integration Tests**:
```rust
// tests/client_integration.rs

#[tokio::test]
async fn test_client_with_mock_coordinator() {
    let mock = MockCoordinator::start().await;
    let client = DoraClient::connect(&mock.addr()).await.unwrap();

    // Verify handshake
    assert!(client.is_connected());

    mock.shutdown().await;
}

#[tokio::test]
async fn test_client_handles_malformed_response() {
    let mock = MockCoordinator::start().await;
    mock.set_response(b"invalid json{{{");

    let mut client = DoraClient::connect(&mock.addr()).await.unwrap();
    let result = client.list_dataflows().await;

    assert!(result.is_err());
    assert!(matches!(result.unwrap_err(), ClientError::ParseError(_)));
}
```

---

### C-03: Dataflow Operations

**Unit Tests**:
```rust
// dora-studio-client/src/client.rs tests

#[cfg(test)]
mod dataflow_tests {
    use super::*;

    #[test]
    fn test_dataflow_entry_serialization() {
        let entry = DataflowEntry {
            uuid: Uuid::new_v4(),
            name: Some("test-flow".to_string()),
            status: DataflowStatus::Running,
            node_count: 4,
            created_at: Utc::now(),
        };

        let json = serde_json::to_string(&entry).unwrap();
        let parsed: DataflowEntry = serde_json::from_str(&json).unwrap();

        assert_eq!(entry.uuid, parsed.uuid);
        assert_eq!(entry.name, parsed.name);
    }

    #[test]
    fn test_dataflow_status_variants() {
        let running = DataflowStatus::Running;
        let finished = DataflowStatus::Finished;
        let failed = DataflowStatus::Failed("OOM".to_string());

        assert!(matches!(running, DataflowStatus::Running));
        assert!(matches!(finished, DataflowStatus::Finished));
        assert!(matches!(failed, DataflowStatus::Failed(msg) if msg == "OOM"));
    }
}
```

**Integration Tests**:
```rust
// tests/dataflow_integration.rs

#[tokio::test]
async fn test_list_dataflows_empty() {
    let mock = MockCoordinator::start().await;
    mock.set_dataflows(vec![]);

    let mut client = DoraClient::connect(&mock.addr()).await.unwrap();
    let flows = client.list_dataflows().await.unwrap();

    assert!(flows.is_empty());
}

#[tokio::test]
async fn test_list_dataflows_multiple() {
    let mock = MockCoordinator::start().await;
    mock.set_dataflows(vec![
        mock_dataflow("flow-1", DataflowStatus::Running),
        mock_dataflow("flow-2", DataflowStatus::Finished),
    ]);

    let mut client = DoraClient::connect(&mock.addr()).await.unwrap();
    let flows = client.list_dataflows().await.unwrap();

    assert_eq!(flows.len(), 2);
}

#[tokio::test]
async fn test_start_dataflow_success() {
    let mock = MockCoordinator::start().await;
    let expected_uuid = Uuid::new_v4();
    mock.expect_start_returns(expected_uuid);

    let mut client = DoraClient::connect(&mock.addr()).await.unwrap();
    let uuid = client.start_dataflow("test.yaml").await.unwrap();

    assert_eq!(uuid, expected_uuid);
}

#[tokio::test]
async fn test_start_dataflow_file_not_found() {
    let mock = MockCoordinator::start().await;
    mock.expect_start_fails("File not found");

    let mut client = DoraClient::connect(&mock.addr()).await.unwrap();
    let result = client.start_dataflow("nonexistent.yaml").await;

    assert!(result.is_err());
}

#[tokio::test]
async fn test_stop_dataflow_success() {
    let mock = MockCoordinator::start().await;
    let uuid = Uuid::new_v4();
    mock.add_running_dataflow(uuid);

    let mut client = DoraClient::connect(&mock.addr()).await.unwrap();
    let result = client.stop_dataflow(&uuid).await;

    assert!(result.is_ok());
}

#[tokio::test]
async fn test_stop_dataflow_not_found() {
    let mock = MockCoordinator::start().await;

    let mut client = DoraClient::connect(&mock.addr()).await.unwrap();
    let result = client.stop_dataflow(&Uuid::new_v4()).await;

    assert!(result.is_err());
}
```

**E2E Tests** (requires real Dora):
```rust
// tests/e2e/dataflow_e2e.rs

#[tokio::test]
#[ignore] // Run with: cargo test --test e2e -- --ignored
async fn test_full_dataflow_lifecycle() {
    let client = DoraClient::connect("127.0.0.1:53290").await.unwrap();

    // Start
    let uuid = client.start_dataflow("examples/simple.yaml").await.unwrap();

    // Verify running
    let flows = client.list_dataflows().await.unwrap();
    let flow = flows.iter().find(|f| f.uuid == uuid).unwrap();
    assert!(matches!(flow.status, DataflowStatus::Running));

    // Stop
    client.stop_dataflow(&uuid).await.unwrap();

    // Verify stopped
    tokio::time::sleep(Duration::from_secs(1)).await;
    let flows = client.list_dataflows().await.unwrap();
    let flow = flows.iter().find(|f| f.uuid == uuid);
    assert!(flow.is_none() || !matches!(flow.unwrap().status, DataflowStatus::Running));
}
```

---

### C-04: Log Subscription

**Unit Tests**:
```rust
#[test]
fn test_log_message_parsing() {
    let json = r#"{
        "timestamp": "2024-01-15T10:30:00Z",
        "node_id": "yolo",
        "level": "INFO",
        "message": "Processing frame 42"
    }"#;

    let log: LogMessage = serde_json::from_str(json).unwrap();
    assert_eq!(log.node_id, Some("yolo".to_string()));
    assert_eq!(log.level, LogLevel::Info);
}

#[test]
fn test_log_level_ordering() {
    assert!(LogLevel::Error > LogLevel::Warn);
    assert!(LogLevel::Warn > LogLevel::Info);
    assert!(LogLevel::Info > LogLevel::Debug);
    assert!(LogLevel::Debug > LogLevel::Trace);
}
```

**Integration Tests**:
```rust
#[tokio::test]
async fn test_log_subscription_receives_logs() {
    let mock = MockCoordinator::start().await;

    let mut client = DoraClient::connect(&mock.addr()).await.unwrap();
    let mut stream = client.subscribe_logs().await.unwrap();

    // Send test log from mock
    mock.emit_log(LogMessage {
        timestamp: Utc::now(),
        node_id: Some("test-node".to_string()),
        level: LogLevel::Info,
        message: "Test message".to_string(),
        ..Default::default()
    });

    let log = stream.next().await.unwrap();
    assert_eq!(log.message, "Test message");
}

#[tokio::test]
async fn test_log_subscription_handles_disconnect() {
    let mock = MockCoordinator::start().await;

    let mut client = DoraClient::connect(&mock.addr()).await.unwrap();
    let mut stream = client.subscribe_logs().await.unwrap();

    mock.shutdown().await;

    // Stream should end gracefully
    let result = stream.next().await;
    assert!(result.is_none());
}
```

---

### C-05: DataFusion Storage

**Unit Tests**:
```rust
// dora-studio-client/src/storage.rs tests

#[cfg(test)]
mod tests {
    use super::*;
    use tempfile::TempDir;

    #[tokio::test]
    async fn test_storage_initialization() {
        let temp_dir = TempDir::new().unwrap();
        let storage = Storage::new(temp_dir.path()).await.unwrap();

        // Verify directories created
        assert!(temp_dir.path().join("metrics").exists());
        assert!(temp_dir.path().join("logs").exists());
        assert!(temp_dir.path().join("spans").exists());
    }

    #[tokio::test]
    async fn test_insert_and_query_metrics() {
        let temp_dir = TempDir::new().unwrap();
        let storage = Storage::new(temp_dir.path()).await.unwrap();

        let metrics = vec![
            NodeMetrics {
                timestamp: Utc::now(),
                node_id: "test-node".to_string(),
                cpu_percent: 45.5,
                memory_mb: 1024.0,
                ..Default::default()
            }
        ];

        storage.insert_metrics(&metrics).await.unwrap();

        let result = storage.query("SELECT * FROM metrics WHERE node_id = 'test-node'").await.unwrap();
        assert_eq!(result.num_rows(), 1);
    }

    #[tokio::test]
    async fn test_query_empty_table() {
        let temp_dir = TempDir::new().unwrap();
        let storage = Storage::new(temp_dir.path()).await.unwrap();

        let result = storage.query("SELECT * FROM metrics").await.unwrap();
        assert_eq!(result.num_rows(), 0);
    }

    #[tokio::test]
    async fn test_invalid_sql_returns_error() {
        let temp_dir = TempDir::new().unwrap();
        let storage = Storage::new(temp_dir.path()).await.unwrap();

        let result = storage.query("INVALID SQL SYNTAX").await;
        assert!(result.is_err());
    }
}
```

**Integration Tests**:
```rust
#[tokio::test]
async fn test_storage_persistence() {
    let temp_dir = TempDir::new().unwrap();

    // Insert data
    {
        let storage = Storage::new(temp_dir.path()).await.unwrap();
        storage.insert_metrics(&[test_metric()]).await.unwrap();
    }

    // Reopen and verify data persisted
    {
        let storage = Storage::new(temp_dir.path()).await.unwrap();
        let result = storage.query("SELECT COUNT(*) FROM metrics").await.unwrap();
        assert!(result.num_rows() > 0);
    }
}

#[tokio::test]
async fn test_storage_time_range_query() {
    let temp_dir = TempDir::new().unwrap();
    let storage = Storage::new(temp_dir.path()).await.unwrap();

    let now = Utc::now();
    let metrics = vec![
        metric_at(now - Duration::hours(2)),
        metric_at(now - Duration::hours(1)),
        metric_at(now),
    ];

    storage.insert_metrics(&metrics).await.unwrap();

    // Query last hour only
    let result = storage.query(&format!(
        "SELECT * FROM metrics WHERE timestamp > '{}'",
        (now - Duration::hours(1)).to_rfc3339()
    )).await.unwrap();

    assert_eq!(result.num_rows(), 2); // Last 2 metrics
}
```

---

### C-06: Parquet Schemas

**Unit Tests**:
```rust
#[test]
fn test_metrics_schema_fields() {
    let schema = metrics_schema();

    assert!(schema.field_with_name("timestamp").is_ok());
    assert!(schema.field_with_name("node_id").is_ok());
    assert!(schema.field_with_name("cpu_percent").is_ok());
    assert!(schema.field_with_name("memory_mb").is_ok());
}

#[test]
fn test_logs_schema_fields() {
    let schema = logs_schema();

    assert!(schema.field_with_name("timestamp").is_ok());
    assert!(schema.field_with_name("level").is_ok());
    assert!(schema.field_with_name("message").is_ok());
}

#[test]
fn test_spans_schema_fields() {
    let schema = spans_schema();

    assert!(schema.field_with_name("trace_id").is_ok());
    assert!(schema.field_with_name("span_id").is_ok());
    assert!(schema.field_with_name("duration_us").is_ok());
}

#[test]
fn test_write_parquet_file() {
    let temp_dir = TempDir::new().unwrap();
    let path = temp_dir.path().join("test.parquet");

    let batch = create_test_record_batch();
    write_parquet(&path, &batch).unwrap();

    assert!(path.exists());
}

#[test]
fn test_read_parquet_file() {
    let temp_dir = TempDir::new().unwrap();
    let path = temp_dir.path().join("test.parquet");

    let original = create_test_record_batch();
    write_parquet(&path, &original).unwrap();

    let loaded = read_parquet(&path).unwrap();
    assert_eq!(original.num_rows(), loaded.num_rows());
}
```

---

## Phase 2: Mini-Apps Tests

### App A: Dataflow Manager

**Unit Tests**:
```rust
// apps/dataflow-manager/tests/unit.rs

#[test]
fn test_dataflow_row_model() {
    let row = DataflowRow {
        uuid: Uuid::new_v4(),
        name: "test".to_string(),
        status: DataflowStatus::Running,
        node_count: 4,
        cpu_percent: 45.0,
        memory_mb: 2048.0,
        expanded: false,
    };

    assert!(!row.expanded);
    assert_eq!(row.status_color(), ACCENT_GREEN);
}

#[test]
fn test_dataflow_row_status_colors() {
    let running = DataflowRow { status: DataflowStatus::Running, ..Default::default() };
    let finished = DataflowRow { status: DataflowStatus::Finished, ..Default::default() };
    let failed = DataflowRow { status: DataflowStatus::Failed("".into()), ..Default::default() };

    assert_eq!(running.status_color(), ACCENT_GREEN);
    assert_eq!(finished.status_color(), TEXT_SECONDARY);
    assert_eq!(failed.status_color(), ACCENT_RED);
}

#[test]
fn test_dataflow_sorting() {
    let mut rows = vec![
        DataflowRow { name: "zebra".into(), ..Default::default() },
        DataflowRow { name: "alpha".into(), ..Default::default() },
        DataflowRow { name: "beta".into(), ..Default::default() },
    ];

    rows.sort_by(|a, b| a.name.cmp(&b.name));

    assert_eq!(rows[0].name, "alpha");
    assert_eq!(rows[2].name, "zebra");
}
```

**Integration Tests**:
```rust
// apps/dataflow-manager/tests/integration.rs

#[tokio::test]
async fn test_dataflow_list_loads_from_client() {
    let mock = MockDoraClient::new();
    mock.set_dataflows(vec![
        mock_dataflow("flow-1"),
        mock_dataflow("flow-2"),
    ]);

    let state = DataflowManagerState::new(mock);
    state.refresh().await.unwrap();

    assert_eq!(state.dataflows().len(), 2);
}

#[tokio::test]
async fn test_start_action_calls_client() {
    let mock = MockDoraClient::new();
    let state = DataflowManagerState::new(mock.clone());

    state.start_dataflow("test.yaml").await.unwrap();

    assert!(mock.was_called("start_dataflow"));
}

#[tokio::test]
async fn test_stop_action_requires_selection() {
    let mock = MockDoraClient::new();
    let state = DataflowManagerState::new(mock);

    // No selection
    let result = state.stop_selected().await;
    assert!(result.is_err());
}
```

**UI Tests**:
```rust
#[test]
fn test_dataflow_table_renders() {
    let mut cx = TestCx::new();
    let widget = DataflowTable::new(&mut cx);

    widget.set_data(vec![mock_dataflow("test")]);
    widget.draw(&mut cx);

    // Verify table renders without panic
    assert!(cx.render_succeeded());
}

#[test]
fn test_row_expansion_toggle() {
    let mut cx = TestCx::new();
    let widget = DataflowTable::new(&mut cx);
    widget.set_data(vec![mock_dataflow("test")]);

    // Click expand button
    cx.simulate_click(widget.row(0).expand_button());

    assert!(widget.row(0).is_expanded());
}
```

---

### App B: YAML Editor

**Unit Tests**:
```rust
// apps/yaml-editor/tests/unit.rs

#[test]
fn test_parse_simple_dataflow() {
    let yaml = r#"
nodes:
  - id: camera
    path: dora-webcam
    outputs: [image]
"#;

    let graph = parse_dataflow_yaml(yaml).unwrap();
    assert_eq!(graph.nodes.len(), 1);
    assert_eq!(graph.nodes[0].id, "camera");
}

#[test]
fn test_parse_dataflow_with_connections() {
    let yaml = r#"
nodes:
  - id: camera
    path: dora-webcam
    outputs: [image]
  - id: detector
    path: dora-yolo
    inputs:
      image: camera/image
"#;

    let graph = parse_dataflow_yaml(yaml).unwrap();
    assert_eq!(graph.edges.len(), 1);
    assert_eq!(graph.edges[0].from, ("camera", "image"));
    assert_eq!(graph.edges[0].to, ("detector", "image"));
}

#[test]
fn test_parse_invalid_yaml() {
    let yaml = "invalid: yaml: syntax::: broken";
    let result = parse_dataflow_yaml(yaml);
    assert!(result.is_err());
}

#[test]
fn test_validation_missing_path() {
    let yaml = r#"
nodes:
  - id: camera
    outputs: [image]
"#;

    let errors = validate_dataflow_yaml(yaml);
    assert!(!errors.is_empty());
    assert!(errors[0].message.contains("path"));
}

#[test]
fn test_validation_duplicate_node_id() {
    let yaml = r#"
nodes:
  - id: node1
    path: test
  - id: node1
    path: test2
"#;

    let errors = validate_dataflow_yaml(yaml);
    assert!(errors.iter().any(|e| e.message.contains("duplicate")));
}

#[test]
fn test_validation_undefined_input() {
    let yaml = r#"
nodes:
  - id: detector
    path: dora-yolo
    inputs:
      image: nonexistent/image
"#;

    let errors = validate_dataflow_yaml(yaml);
    assert!(errors.iter().any(|e| e.message.contains("undefined")));
}
```

**Integration Tests** (with makepad-flow):
```rust
#[tokio::test]
async fn test_yaml_change_updates_graph() {
    let mut editor = YamlEditorState::new();

    editor.set_yaml(r#"
nodes:
  - id: camera
    path: dora-webcam
"#);

    // Wait for debounced update
    tokio::time::sleep(Duration::from_millis(500)).await;

    assert_eq!(editor.graph().nodes.len(), 1);
}

#[test]
fn test_graph_layout_algorithm() {
    let graph = DataflowGraph {
        nodes: vec![
            Node { id: "a".into(), .. },
            Node { id: "b".into(), .. },
            Node { id: "c".into(), .. },
        ],
        edges: vec![
            Edge { from: "a", to: "b" },
            Edge { from: "b", to: "c" },
        ],
    };

    let layout = compute_layout(&graph);

    // Verify hierarchical layout
    assert!(layout.position("a").y < layout.position("b").y);
    assert!(layout.position("b").y < layout.position("c").y);
}
```

---

### App C: Log Viewer

**Unit Tests**:
```rust
// apps/log-viewer/tests/unit.rs

#[test]
fn test_log_filter_by_level() {
    let logs = vec![
        log_entry(LogLevel::Debug, "debug msg"),
        log_entry(LogLevel::Info, "info msg"),
        log_entry(LogLevel::Warn, "warn msg"),
        log_entry(LogLevel::Error, "error msg"),
    ];

    let filter = LogFilter { min_level: LogLevel::Warn, ..Default::default() };
    let filtered: Vec<_> = logs.iter().filter(|l| filter.matches(l)).collect();

    assert_eq!(filtered.len(), 2);
}

#[test]
fn test_log_filter_by_node() {
    let logs = vec![
        log_entry_node("camera", "msg1"),
        log_entry_node("yolo", "msg2"),
        log_entry_node("camera", "msg3"),
    ];

    let filter = LogFilter { node_id: Some("camera".into()), ..Default::default() };
    let filtered: Vec<_> = logs.iter().filter(|l| filter.matches(l)).collect();

    assert_eq!(filtered.len(), 2);
}

#[test]
fn test_log_filter_by_text_regex() {
    let logs = vec![
        log_entry(LogLevel::Info, "Processing frame 42"),
        log_entry(LogLevel::Info, "Error in processing"),
        log_entry(LogLevel::Info, "Frame 43 complete"),
    ];

    let filter = LogFilter {
        text_pattern: Some(Regex::new(r"frame \d+").unwrap()),
        ..Default::default()
    };
    let filtered: Vec<_> = logs.iter().filter(|l| filter.matches(l)).collect();

    assert_eq!(filtered.len(), 2);
}

#[test]
fn test_log_filter_combined() {
    let logs = vec![
        LogMessage { level: LogLevel::Error, node_id: Some("camera".into()), message: "Error A".into(), .. },
        LogMessage { level: LogLevel::Error, node_id: Some("yolo".into()), message: "Error B".into(), .. },
        LogMessage { level: LogLevel::Info, node_id: Some("camera".into()), message: "Info C".into(), .. },
    ];

    let filter = LogFilter {
        min_level: LogLevel::Error,
        node_id: Some("camera".into()),
        ..Default::default()
    };
    let filtered: Vec<_> = logs.iter().filter(|l| filter.matches(l)).collect();

    assert_eq!(filtered.len(), 1);
    assert_eq!(filtered[0].message, "Error A");
}
```

**Performance Tests**:
```rust
#[test]
fn test_virtualized_list_performance() {
    let logs: Vec<_> = (0..100_000).map(|i| log_entry(LogLevel::Info, &format!("Log {}", i))).collect();

    let list = VirtualizedLogList::new();
    list.set_data(logs);

    // Measure scroll performance
    let start = Instant::now();
    for offset in (0..100_000).step_by(1000) {
        list.scroll_to(offset);
        list.visible_rows(); // Force render calculation
    }
    let elapsed = start.elapsed();

    assert!(elapsed < Duration::from_secs(1), "Scrolling too slow: {:?}", elapsed);
}

#[test]
fn test_filter_performance_100k_logs() {
    let logs: Vec<_> = (0..100_000).map(|i| {
        log_entry(
            if i % 10 == 0 { LogLevel::Error } else { LogLevel::Info },
            &format!("Log message {}", i)
        )
    }).collect();

    let filter = LogFilter { min_level: LogLevel::Error, ..Default::default() };

    let start = Instant::now();
    let filtered: Vec<_> = logs.iter().filter(|l| filter.matches(l)).collect();
    let elapsed = start.elapsed();

    assert_eq!(filtered.len(), 10_000);
    assert!(elapsed < Duration::from_millis(100), "Filtering too slow: {:?}", elapsed);
}
```

---

### App D: Telemetry Dashboard

**Unit Tests**:
```rust
// apps/telemetry-dashboard/tests/unit.rs

#[test]
fn test_metrics_aggregation_avg() {
    let metrics = vec![
        Metric { value: 10.0, .. },
        Metric { value: 20.0, .. },
        Metric { value: 30.0, .. },
    ];

    let avg = aggregate_metrics(&metrics, Aggregation::Avg);
    assert!((avg - 20.0).abs() < 0.001);
}

#[test]
fn test_metrics_aggregation_percentile() {
    let mut metrics: Vec<_> = (1..=100).map(|i| Metric { value: i as f64, .. }).collect();

    let p50 = calculate_percentile(&metrics, 50.0);
    let p95 = calculate_percentile(&metrics, 95.0);
    let p99 = calculate_percentile(&metrics, 99.0);

    assert!((p50 - 50.0).abs() < 1.0);
    assert!((p95 - 95.0).abs() < 1.0);
    assert!((p99 - 99.0).abs() < 1.0);
}

#[test]
fn test_time_range_5m() {
    let range = TimeRange::Minutes(5);
    let now = Utc::now();

    let (start, end) = range.to_bounds(now);

    assert_eq!(end, now);
    assert_eq!(start, now - Duration::minutes(5));
}

#[test]
fn test_chart_data_downsampling() {
    // 10000 points should be downsampled for chart display
    let data: Vec<_> = (0..10000).map(|i| DataPoint { x: i as f64, y: (i % 100) as f64 }).collect();

    let downsampled = downsample_for_chart(&data, 500);

    assert!(downsampled.len() <= 500);
    // Should preserve min/max in each bucket
}
```

**Integration Tests** (with makepad-chart):
```rust
#[tokio::test]
async fn test_chart_updates_on_time_range_change() {
    let mock_storage = MockStorage::new();
    mock_storage.set_metrics(generate_test_metrics(Duration::hours(24)));

    let dashboard = TelemetryDashboardState::new(mock_storage);

    dashboard.set_time_range(TimeRange::Hours(1)).await;
    let data_1h = dashboard.cpu_chart_data();

    dashboard.set_time_range(TimeRange::Hours(24)).await;
    let data_24h = dashboard.cpu_chart_data();

    assert!(data_24h.len() > data_1h.len());
}

#[test]
fn test_golden_signals_calculation() {
    let metrics = GoldenSignalsInput {
        request_count: 1000,
        error_count: 10,
        latencies_ms: vec![10.0, 20.0, 30.0, 100.0, 200.0],
    };

    let signals = calculate_golden_signals(&metrics);

    assert_eq!(signals.request_rate, 1000);
    assert!((signals.error_rate - 1.0).abs() < 0.1); // 1%
    assert!(signals.p50_latency < signals.p95_latency);
    assert!(signals.p95_latency < signals.p99_latency);
}
```

---

## Phase 3: AI Agent Tests

### AI-01: LlmClient Trait

**Unit Tests**:
```rust
#[test]
fn test_tool_definition_serialization() {
    let tool = ToolDefinition {
        name: "list_dataflows".to_string(),
        description: "List all dataflows".to_string(),
        parameters: json!({
            "type": "object",
            "properties": {}
        }),
    };

    let json = serde_json::to_string(&tool).unwrap();
    assert!(json.contains("list_dataflows"));
}

#[test]
fn test_agent_response_variants() {
    let text = AgentResponse::Text("Hello".to_string());
    let tool_call = AgentResponse::ToolCall(ToolCall {
        id: "123".to_string(),
        name: "test".to_string(),
        arguments: json!({}),
    });

    assert!(matches!(text, AgentResponse::Text(_)));
    assert!(matches!(tool_call, AgentResponse::ToolCall(_)));
}
```

---

### AI-04: AgentCoordinator

**Unit Tests**:
```rust
#[tokio::test]
async fn test_agent_simple_response() {
    let mock_llm = MockLlmClient::new();
    mock_llm.set_response(AgentResponse::Text("Hello!".to_string()));

    let coordinator = AgentCoordinator::new(mock_llm);
    let response = coordinator.process_message("dataflow-manager", "Hello").await.unwrap();

    assert!(matches!(response, AgentResponse::Text(s) if s == "Hello!"));
}

#[tokio::test]
async fn test_agent_tool_call_execution() {
    let mock_llm = MockLlmClient::new();
    mock_llm.set_response(AgentResponse::ToolCall(ToolCall {
        id: "1".to_string(),
        name: "list_dataflows".to_string(),
        arguments: json!({}),
    }));
    mock_llm.set_continuation(AgentResponse::Text("Found 2 dataflows".to_string()));

    let coordinator = AgentCoordinator::new(mock_llm);
    coordinator.register_tool("list_dataflows", |_| async { Ok(json!(["flow1", "flow2"])) });

    let response = coordinator.process_message("dataflow-manager", "List dataflows").await.unwrap();

    assert!(matches!(response, AgentResponse::Text(s) if s.contains("2 dataflows")));
}

#[tokio::test]
async fn test_agent_unknown_tool_error() {
    let mock_llm = MockLlmClient::new();
    mock_llm.set_response(AgentResponse::ToolCall(ToolCall {
        id: "1".to_string(),
        name: "unknown_tool".to_string(),
        arguments: json!({}),
    }));

    let coordinator = AgentCoordinator::new(mock_llm);
    let result = coordinator.process_message("test", "Do something").await;

    assert!(result.is_err());
}
```

**Integration Tests**:
```rust
#[tokio::test]
async fn test_agent_with_real_tools() {
    let mock_llm = MockLlmClient::new();
    let mock_client = MockDoraClient::new();
    mock_client.set_dataflows(vec![mock_dataflow("test-flow")]);

    let coordinator = AgentCoordinator::new(mock_llm);
    coordinator.register_dataflow_tools(mock_client);

    // Simulate tool call
    let tool_result = coordinator.execute_tool(&ToolCall {
        id: "1".to_string(),
        name: "list_dataflows".to_string(),
        arguments: json!({}),
    }).await.unwrap();

    assert!(tool_result.to_string().contains("test-flow"));
}
```

---

### AI-05: Tools Per Mini-App

**Unit Tests** (per tool):
```rust
// Dataflow Manager Tools
#[tokio::test]
async fn test_tool_list_dataflows() {
    let mock_client = MockDoraClient::new();
    mock_client.set_dataflows(vec![
        mock_dataflow("flow-1"),
        mock_dataflow("flow-2"),
    ]);

    let result = tool_list_dataflows(&mock_client, json!({})).await.unwrap();

    let flows: Vec<DataflowInfo> = serde_json::from_value(result).unwrap();
    assert_eq!(flows.len(), 2);
}

#[tokio::test]
async fn test_tool_start_dataflow() {
    let mock_client = MockDoraClient::new();
    let expected_uuid = Uuid::new_v4();
    mock_client.expect_start_returns(expected_uuid);

    let result = tool_start_dataflow(&mock_client, json!({
        "yaml_path": "test.yaml"
    })).await.unwrap();

    assert!(result["uuid"].as_str().unwrap() == expected_uuid.to_string());
}

#[tokio::test]
async fn test_tool_start_dataflow_missing_path() {
    let mock_client = MockDoraClient::new();

    let result = tool_start_dataflow(&mock_client, json!({})).await;

    assert!(result.is_err());
}

// YAML Editor Tools
#[tokio::test]
async fn test_tool_validate_yaml_valid() {
    let result = tool_validate_yaml(json!({
        "yaml": "nodes:\n  - id: test\n    path: test-path"
    })).await.unwrap();

    assert!(result["valid"].as_bool().unwrap());
    assert!(result["errors"].as_array().unwrap().is_empty());
}

#[tokio::test]
async fn test_tool_validate_yaml_invalid() {
    let result = tool_validate_yaml(json!({
        "yaml": "nodes:\n  - id: test"  // Missing path
    })).await.unwrap();

    assert!(!result["valid"].as_bool().unwrap());
    assert!(!result["errors"].as_array().unwrap().is_empty());
}

#[tokio::test]
async fn test_tool_generate_dataflow() {
    let result = tool_generate_dataflow(json!({
        "description": "webcam to YOLO detection",
        "nodes": ["webcam", "yolo", "display"]
    })).await.unwrap();

    let yaml = result["yaml"].as_str().unwrap();
    assert!(yaml.contains("webcam"));
    assert!(yaml.contains("yolo"));
}

// Log Viewer Tools
#[tokio::test]
async fn test_tool_search_logs() {
    let mock_storage = MockStorage::new();
    mock_storage.set_logs(vec![
        log_entry(LogLevel::Error, "Error in camera"),
        log_entry(LogLevel::Info, "Processing complete"),
    ]);

    let result = tool_search_logs(&mock_storage, json!({
        "pattern": "Error",
        "level": "ERROR"
    })).await.unwrap();

    let logs: Vec<LogEntry> = serde_json::from_value(result["logs"].clone()).unwrap();
    assert_eq!(logs.len(), 1);
}

#[tokio::test]
async fn test_tool_analyze_logs() {
    let mock_storage = MockStorage::new();
    mock_storage.set_logs(generate_sample_logs(100));

    let result = tool_analyze_logs(&mock_storage, json!({
        "time_range": "1h"
    })).await.unwrap();

    assert!(result["summary"].is_string());
    assert!(result["error_count"].is_number());
    assert!(result["patterns"].is_array());
}

// Telemetry Dashboard Tools
#[tokio::test]
async fn test_tool_find_bottleneck() {
    let mock_storage = MockStorage::new();
    mock_storage.set_metrics(vec![
        node_metric("camera", 20.0, 128.0),
        node_metric("yolo", 98.0, 2048.0),  // High CPU
        node_metric("plot", 5.0, 64.0),
    ]);

    let result = tool_find_bottleneck(&mock_storage, json!({
        "dataflow_id": "test-flow"
    })).await.unwrap();

    assert_eq!(result["bottleneck_node"].as_str().unwrap(), "yolo");
    assert!(result["reason"].as_str().unwrap().contains("CPU"));
}
```

---

### AI-06: ChatBar Widget

**UI Tests**:
```rust
#[test]
fn test_chatbar_renders() {
    let mut cx = TestCx::new();
    let widget = ChatBar::new(&mut cx);
    widget.draw(&mut cx);

    assert!(cx.render_succeeded());
}

#[test]
fn test_chatbar_input_handling() {
    let mut cx = TestCx::new();
    let widget = ChatBar::new(&mut cx);

    cx.simulate_text_input("Hello AI");

    assert_eq!(widget.input_text(), "Hello AI");
}

#[test]
fn test_chatbar_submit_on_enter() {
    let mut cx = TestCx::new();
    let mut submitted = false;
    let widget = ChatBar::new(&mut cx);
    widget.on_submit(|msg| submitted = true);

    cx.simulate_text_input("Test message");
    cx.simulate_key_press(KeyCode::Return);

    assert!(submitted);
}

#[test]
fn test_chatbar_shows_response() {
    let mut cx = TestCx::new();
    let widget = ChatBar::new(&mut cx);

    widget.set_response("AI response here");
    widget.draw(&mut cx);

    assert!(cx.rendered_text_contains("AI response here"));
}

#[test]
fn test_chatbar_loading_state() {
    let mut cx = TestCx::new();
    let widget = ChatBar::new(&mut cx);

    widget.set_loading(true);
    widget.draw(&mut cx);

    assert!(widget.is_input_disabled());
    assert!(cx.rendered_contains_indicator());
}
```

---

## Phase 4: Polish Tests

### P-01: Unit Test Coverage

**Coverage Requirements**:
```toml
# .cargo/config.toml
[env]
CARGO_INCREMENTAL = "0"
RUSTFLAGS = "-Cinstrument-coverage"
LLVM_PROFILE_FILE = "coverage-%p-%m.profraw"
```

```bash
# Generate coverage report
cargo test
grcov . -s . --binary-path ./target/debug/ -t html --branch --ignore-not-existing -o ./coverage/

# Check coverage threshold
grcov . -s . --binary-path ./target/debug/ -t covdir --branch | jq '.coveragePercent > 70'
```

---

### P-03: Performance Tests

**Benchmarks**:
```rust
// benches/performance.rs

use criterion::{criterion_group, criterion_main, Criterion, BenchmarkId};

fn bench_log_filtering(c: &mut Criterion) {
    let logs: Vec<_> = (0..100_000).map(|i| log_entry(LogLevel::Info, &format!("Log {}", i))).collect();

    c.bench_function("filter_100k_logs", |b| {
        let filter = LogFilter { min_level: LogLevel::Warn, ..Default::default() };
        b.iter(|| {
            logs.iter().filter(|l| filter.matches(l)).count()
        })
    });
}

fn bench_chart_rendering(c: &mut Criterion) {
    let data: Vec<_> = (0..10_000).map(|i| DataPoint { x: i as f64, y: (i % 100) as f64 }).collect();

    c.bench_function("render_chart_10k_points", |b| {
        b.iter(|| {
            downsample_for_chart(&data, 500)
        })
    });
}

fn bench_yaml_parsing(c: &mut Criterion) {
    let yaml = generate_large_dataflow_yaml(100); // 100 nodes

    c.bench_function("parse_100_node_yaml", |b| {
        b.iter(|| {
            parse_dataflow_yaml(&yaml)
        })
    });
}

fn bench_storage_query(c: &mut Criterion) {
    let rt = tokio::runtime::Runtime::new().unwrap();
    let storage = rt.block_on(setup_test_storage_with_data(100_000));

    c.bench_function("query_metrics_100k", |b| {
        b.to_async(&rt).iter(|| async {
            storage.query("SELECT * FROM metrics WHERE cpu_percent > 50").await
        })
    });
}

criterion_group!(benches, bench_log_filtering, bench_chart_rendering, bench_yaml_parsing, bench_storage_query);
criterion_main!(benches);
```

**Performance Assertions**:
```rust
#[test]
fn test_startup_time() {
    let start = Instant::now();
    let _app = App::new();
    let elapsed = start.elapsed();

    assert!(elapsed < Duration::from_secs(1), "Startup too slow: {:?}", elapsed);
}

#[test]
fn test_memory_usage() {
    let before = get_memory_usage();
    let app = App::new();
    app.load_sample_data();
    let after = get_memory_usage();

    let diff_mb = (after - before) / (1024 * 1024);
    assert!(diff_mb < 100, "Memory usage too high: {}MB", diff_mb);
}
```

---

## Test Utilities

### Mock Implementations

```rust
// tests/mocks/mod.rs

pub struct MockDoraClient {
    dataflows: Arc<Mutex<Vec<DataflowEntry>>>,
    call_log: Arc<Mutex<Vec<String>>>,
}

impl MockDoraClient {
    pub fn new() -> Self { ... }
    pub fn set_dataflows(&self, flows: Vec<DataflowEntry>) { ... }
    pub fn was_called(&self, method: &str) -> bool { ... }
}

pub struct MockLlmClient {
    responses: Arc<Mutex<VecDeque<AgentResponse>>>,
}

impl MockLlmClient {
    pub fn new() -> Self { ... }
    pub fn set_response(&self, response: AgentResponse) { ... }
    pub fn set_continuation(&self, response: AgentResponse) { ... }
}

pub struct MockStorage {
    metrics: Arc<Mutex<Vec<NodeMetrics>>>,
    logs: Arc<Mutex<Vec<LogMessage>>>,
}

impl MockStorage {
    pub fn new() -> Self { ... }
    pub fn set_metrics(&self, metrics: Vec<NodeMetrics>) { ... }
    pub fn set_logs(&self, logs: Vec<LogMessage>) { ... }
}
```

### Test Fixtures

```rust
// tests/fixtures/mod.rs

pub fn mock_dataflow(name: &str) -> DataflowEntry {
    DataflowEntry {
        uuid: Uuid::new_v4(),
        name: Some(name.to_string()),
        status: DataflowStatus::Running,
        node_count: 4,
        created_at: Utc::now(),
    }
}

pub fn log_entry(level: LogLevel, message: &str) -> LogMessage {
    LogMessage {
        timestamp: Utc::now(),
        level,
        message: message.to_string(),
        ..Default::default()
    }
}

pub fn node_metric(node_id: &str, cpu: f32, memory: f64) -> NodeMetrics {
    NodeMetrics {
        timestamp: Utc::now(),
        node_id: node_id.to_string(),
        cpu_percent: cpu,
        memory_mb: memory,
        ..Default::default()
    }
}

pub fn generate_sample_logs(count: usize) -> Vec<LogMessage> {
    (0..count).map(|i| {
        log_entry(
            match i % 4 {
                0 => LogLevel::Debug,
                1 => LogLevel::Info,
                2 => LogLevel::Warn,
                _ => LogLevel::Error,
            },
            &format!("Sample log message {}", i)
        )
    }).collect()
}
```

---

## Running Tests

```bash
# Run all tests
cargo test --workspace

# Run specific phase tests
cargo test --package dora-studio-client
cargo test --package dataflow-manager

# Run integration tests only
cargo test --test integration

# Run E2E tests (requires running Dora)
cargo test --test e2e -- --ignored

# Run with coverage
cargo tarpaulin --out Html

# Run benchmarks
cargo bench

# Run specific benchmark
cargo bench --bench performance -- log_filtering
```

---

## CI/CD Integration

```yaml
# .github/workflows/test.yml

name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable

      - name: Run unit tests
        run: cargo test --workspace

      - name: Run integration tests
        run: cargo test --test integration

      - name: Check coverage
        run: |
          cargo tarpaulin --out Xml
          # Fail if coverage < 70%

      - name: Run benchmarks (compare)
        run: cargo bench -- --save-baseline pr
```

---

## Summary

| Phase | Unit Tests | Integration Tests | E2E Tests |
|-------|------------|-------------------|-----------|
| Phase 0 | 15 | 5 | 0 |
| Phase 1 | 40 | 25 | 5 |
| Phase 2 | 80 | 40 | 10 |
| Phase 3 | 35 | 15 | 5 |
| Phase 4 | 10 | 5 | 0 |
| **Total** | **180** | **90** | **20** |
