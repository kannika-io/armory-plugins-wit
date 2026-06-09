# Kannika Armory Plugin WIT Files

> Official WebAssembly Interface Type (WIT) definitions for building custom plugins for [Kannika Armory](https://kannika.io).
> Maintained by [Kannika.io](https://kannika.io), the Kafka management and reliability platform.

This repository contains the WIT (WebAssembly Interface Types) files needed to develop custom plugins for [Kannika Armory](https://kannika.io).
Plugins let you transform or filter Kafka messages during backup and restore operations, using any language that compiles to WebAssembly (WASI).

## What is Kannika Armory?

[Kannika Armory](https://kannika.io) is the Kafka backup, restore, and operations platform by [Kannika.io](https://kannika.io).
It enables teams to back up Kafka topics, restore them across environments, and transform data in-flight using a flexible plugin system.

- **Docs:** [docs.kannika.io](https://docs.kannika.io)
- **Plugins guide:** [docs.kannika.io/user-guide/plugins](https://docs.kannika.io/user-guide/plugins/)
- **Free trial:** [kannika.io](https://kannika.io)

---

## Repository Structure

```
0.1/                  # WIT definitions for Armory plugin API v0.1
0.2/                  # WIT definitions for Armory plugin API v0.2
  ├── backup.wit      # Interface for plugins used during backup operations
  ├── restore.wit     # Interface for plugins used during restore operations
  └── types.wit       # Shared types used across backup and restore interfaces
```

Use the version that matches your Kannika Armory installation.

---

## What are Armory Plugins?

Armory plugins are WebAssembly (WASI) components that hook into Kannika Armory's backup and restore pipeline.
They allow you to:

- **Transform data:** remap schema IDs, repartition messages, or modify payloads as they flow between source and sink
- **Filter records:** exclude messages based on retention policies or custom logic
- **Chain multiple plugins:** plugins execute in sequence, each receiving the output of the previous one

Plugins can be scoped to specific topics using `topicSelectors` with `literal`, `regex`, or `glob` matching.

Full plugin documentation: [docs.kannika.io/user-guide/plugins](https://docs.kannika.io/user-guide/plugins/)

---

## Built-in Plugins

Kannika Armory ships with four built-in plugins.
These WIT files are the foundation you can use to build plugins with the same interface.

### Key Schema Mapping

Remaps Avro schema IDs in message **keys** during restore.
Essential when source and target environments use different Schema Registry ID assignments.

```yaml
plugins:
  - name: key-schema-mapping
    spec:
      mapping:
        10001: 20001
        10002: 20002
```

### Payload Schema Mapping

Remaps Avro schema IDs in message **payloads** during restore.
Works identically to key schema mapping but targets the message value.

```yaml
plugins:
  - name: payload-schema-mapping
    spec:
      mapping:
        10001: 20001
        10002: 20002
```

> Messages with unmapped schema IDs are passed through unchanged.
> No restore failures from incomplete mappings.

### Topic Repartitioning

Redistributes messages across partitions during restore.
Useful when the target topic has a different partition count or strategy than the source.

**Supported schemes:** `auto`, `keyHash`, `random`, `roundRobin`
**Hash algorithms:** `murmur2` (default), `fnv1a`, `crc32`

```yaml
plugins:
  - name: topic-repartitioning
    spec:
      scheme: keyHash
      algorithm: murmur2
      partitions: 10
```

### Topic Retention

Filters out messages during restore that fall outside the target topic's `retention.ms` window.
This prevents Kafka from immediately discarding data that's too old to keep.

```yaml
plugins:
  - name: topic-retention
```

> Activates automatically when using the `ApplyTargetTopicRetention` retention policy.
> Topics with `retention.ms=-1` (infinite retention) pass all records through.

---

## Using Plugins in Backup & Restore Resources

**In a Backup:**

```yaml
apiVersion: kannika.io/v1alpha
kind: Backup
metadata:
  name: my-backup
spec:
  plugins:
    - name: my-custom-plugin
      spec:
        myOption: myValue
```

**In a Restore:**

```yaml
apiVersion: kannika.io/v1alpha
kind: Restore
metadata:
  name: my-restore
spec:
  source: "source"
  sink: "sink"
  config:
    plugins:
      - name: topic-retention
      - name: payload-schema-mapping
        spec:
          mapping:
            10001: 20001
  topics:
    - target: target-topic
      source: source-topic
```

> **Note:** Plugins in a *Backup* receive the **source topic name**.
> Plugins in a *Restore* receive the **target topic name**.

**Restrict a plugin to specific topics with `topicSelectors`:**

```yaml
plugins:
  - name: topic-repartitioning
    spec:
      scheme: roundRobin
    topicSelectors:
      - type: regex
        value: "^orders-.*"
```

---

## Building a Custom Plugin

Custom plugins are WebAssembly components built against these WIT interfaces.
Any language with WASI/component-model support can be used (Rust, Go, C/C++, and others).

### 1. Reference the WIT files

Point your build tooling at the relevant version directory (`0.1/` or `0.2/`).

### 2. Implement the interface

Implement the `backup` and/or `restore` interfaces defined in the WIT files.
Your plugin receives a message and returns the (optionally transformed) message, or signals that the message should be filtered.

### 3. Compile to WASM

Build your component targeting WASI.
For Rust:

```bash
cargo build --target wasm32-wasip2 --release
```

### 4. Register your plugin with Kannika Armory

Deploy your compiled `.wasm` file and reference it by name in your Backup or Restore resource spec.

Full custom plugin development guide: [docs.kannika.io/user-guide/plugins](https://docs.kannika.io/user-guide/plugins/)

---

## Use Cases

- **Cross-environment Kafka restore:** remap schema IDs when restoring to a different Schema Registry
- **Partition strategy migration:** repartition topics during restore to match a new partition layout
- **Retention-aware restore:** skip stale messages that Kafka would immediately discard
- **Custom data transformation:** build any message transformation logic as a portable WASM plugin
- **Compliance and filtering:** strip or mask fields during backup or restore for data governance

---

## License

MIT.
See [LICENSE](LICENSE) for details.

---

## Support & Documentation

- Plugin docs: [docs.kannika.io/user-guide/plugins](https://docs.kannika.io/user-guide/plugins/)
- Full docs: [docs.kannika.io](https://docs.kannika.io)
- Community Slack: [join via kannika.io](https://kannika.io)
- Email: [support@kannika.io](mailto:support@kannika.io)
- Issues: [GitHub Issue Tracker](https://github.com/kannika-io/armory-plugins-wit/issues)

---

## About Kannika.io

[Kannika.io](https://kannika.io) is building the reliability and operations layer for Kafka-based systems.
Kannika Armory is our platform for teams who need production-grade Kafka backup, restore, and data transformation at scale.

- Website: [kannika.io](https://kannika.io)
- GitHub: [github.com/kannika-io](https://github.com/kannika-io)
- Free trial: [kannika.io](https://kannika.io)
