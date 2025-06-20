# GeoBypass-Rules

Rule and proxy configuration files for use with [**GeoBypasser**](https://github.com/MeGaNeKoS/GeoBypasser) — a tool designed to bypass geo-restrictions on various streaming services like **Bilibili**, **Crunchyroll**, and more.

---

## 🗂 Project Structure

All rules are organized by service folder, with each folder containing:

```

\<service\_name>/\<service\_name>-rule.json
\<service\_name>/\<service\_name>-keep-alive.json

```

This structure keeps services cleanly separated and easy to manage.

---

## 📥 How to Import

### 🔄 Keep-Alive Rules

1. Open your **Dashboard**.
2. Go to the **Keep-Alive** section.
3. Click **Import Rules**.
4. Select the appropriate JSON file (e.g., `Bilibili.tv-keep-alive.json`).

### 📜 Rule Files

1. Go to the **Rules** section.
2. Click **Import Rules**.
3. Choose the relevant JSON file (e.g., `Crunchyroll-rule.json`).

### 🌐 Proxy Configuration

1. Go to the **Proxy** section in the Dashboard.
2. Click **Import Rules**.
3. Import the file located at:  
   `community_proxy/us.json`

---

## 🌍 Free Community Proxy

A free proxy configuration is included for public use:

📁 **Location:** `community_proxy/us.json`  
> ⚠️ Please use respectfully. Repeated abuse may force us to limit access or discontinue the proxy for everyone.

---

## 🤝 Contributing

Want to add support for a new service?

1. Follow the existing structure using:
   - `<service_name>/<service_name>-rule.json`
   - `<service_name>/<service_name>-keep-alive.json` (if applicable)
2. Open a **Pull Request (PR)** with your changes.

Community contributions are encouraged and appreciated!

---

## 🪪 License

This project is open-source and maintained by the community.  
Feel free to use, modify, and share—just do so responsibly.

---

Thanks for supporting a respectful and open internet 🙌
