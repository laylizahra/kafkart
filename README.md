# Kafkart: Real-Time Flash Sale System with Apache Kafka on Microsoft Azure 

Kafkart is an online store designed for flash sales, where many buyers compete for a very small stock.
Orders flow through **Apache Kafka** as events, stock is reserved with an atomic SQL update, and
buyers see stock and order status change live in the browser. The whole system runs with
**Docker Compose** and is deployed on a single **Microsoft Azure** virtual machine.

> Small Project, Cloud Computing, Department of Mathematics, Institut Teknologi Sepuluh Nopember (ITS).

## Highlights

- **No overselling:** one atomic `UPDATE ... WHERE stok >= qty` reserves stock.
- **Event-driven pipeline:** 4 Kafka topics, 6 partitions, replication factor 3, `acks=all`.
- **Choreography saga:** a failed payment returns the reserved stock.
- **Horizontal scaling:** add inventory and payment instances; partitions are shared between them.
- **Fault tolerance:** the system keeps working when one broker stops.
- **Live updates:** Server-Sent Events (SSE) for buyers and admins.
- **11 containers** started with one command.

## Architecture

```
Browser -> Gateway (FastAPI) -> [orders] -> Inventory -> [inventory-updates] -> Payment -> [payments]
   ^                                                                                         |
   +------------- SSE <------ Gateway also reads inventory-updates and payments <------------+
                    Notification reads inventory-updates and payments -> order_events (SQL Server)
```

| Topic | Partitions | Producers | Consumers |
|---|---|---|---|
| `orders` | 6 | Gateway, simulator | Inventory |
| `inventory-updates` | 6 | Inventory, admin API | Payment, Gateway, Notification |
| `payments` | 6 | Payment | Inventory, Gateway, Notification |
| `dead-letter` | 3 | Consumer services | Manual inspection |

## Tech stack

| Layer | Technology |
|---|---|
| Web and API | Python, FastAPI, Jinja2, HTML, CSS, vanilla JavaScript |
| Event streaming | Apache Kafka 3.8 (3 brokers, KRaft), `confluent-kafka` |
| Database | SQL Server 2022 |
| Deployment | Docker Compose, Microsoft Azure VM |
| Monitoring | Kafka UI, admin dashboard, Azure Monitor |

## Screenshots

![Shop](docs/screenshots/01-beranda.png)
![Admin dashboard](docs/screenshots/07-admin-dashboard.png)

## Quick start

Requirements: Docker Desktop (or Docker Engine) with at least 6 GB of memory.

```bash
cp .env.example .env        # then edit the passwords and SECRET_KEY
docker compose up -d --build
docker compose ps -a        # wait until Kafka and SQL Server are healthy
```

| Service | Local (Docker on your machine) | Azure deployment |
|---|---|---|
| Shop | http://localhost:8000 | http://kafkart.indonesiacentral.cloudapp.azure.com |
| Admin | http://localhost:8000/admin | http://kafkart.indonesiacentral.cloudapp.azure.com/admin |
| Kafka UI | http://localhost:8080 | Not public. Open an SSH tunnel: `ssh -L 8081:localhost:8080 <user>@<azure-dns-name>`, then http://127.0.0.1:8081 |

> The Azure VM is stopped when it is not in use, so the Azure links work only while the VM is running
> (for example during the demo). On Azure the shop uses port 80 (`WEB_PORT=80`) and plain HTTP.

Create or change the admin account: `python -m tools.create_admin --username admin --password "<password>"`.

## Scale out and test

```bash
docker compose up -d --no-recreate --scale inventory=3 --scale payment=3 inventory payment

pip install -r requirements.txt -r tools/requirements.txt
python -m tools.check_integration
```

## Experiments

```bash
python -m tools.run_experiments scalability --replicas 1 2 3 6 8 --events 1500 --repeats 3
python -m tools.run_experiments producer --messages 50000
python -m tools.fault_test broker
python -m tools.fault_test consumer
python -m tools.plot_results
```

Results are written to `results/`. See [`docs/PENGUJIAN.md`](docs/PENGUJIAN.md) for the test plan.

## Deployment on Azure

Standard D2s_v3 (2 vCPU, 8 GiB), Ubuntu 24.04. Only ports 80 and 22 are open in the network
security group; Kafka UI is reached through an SSH tunnel. Set `WEB_PORT=80` in `.env`.
Stop (deallocate) the VM after use to save credit.

## Project structure

```
app/         gateway (FastAPI), services (inventory, payment, notification), common
database/    SQL schema, views, seed data, init script
kafka/       topic creation script
tools/       integration test, load test, experiments, plots
docs/        architecture, API, database, testing (Indonesian), screenshots
paper/       IEEE-style paper (LaTeX)
```

## Limitations

- All components run on one VM, so it remains a single point of failure.
- Payment is simulated.
- Kafka brokers have no persistent volume: messages are lost if containers are removed.


## Team (Group 5)

Raychana Maharani Zahra, Nur Afiyanti Muzaroh, Chyntia Dwi Rachma, Rifqi Maulana Ilham,
Regan Alif Ardyanto, Annash Ragil Putra Supartono, Layli Nurul Fatimah Zahra.

## License

MIT (see `LICENSE`).
