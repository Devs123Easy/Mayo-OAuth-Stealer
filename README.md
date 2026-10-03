# Malware Analysis Report: Myau Minecraft Session Hijacker

A technical breakdown of the malicious Minecraft mod `Myau-250910+9.jar`. This mod advertises itself as a PvP ghost client/utility mod but functions as a sophisticated **Microsoft OAuth Token Stealer (Session Hijacker)** designed to bypass Two-Factor Authentication (2FA) and permanently steal Minecraft accounts.

## File Information

* **File Name:** `Myau-250910+9.jar`

* **SHA-256 Hash:** `BD341D947DBB1297DBF73A327E25D6A1AA7B960BF56C217E39804CC6E6F61CA1`

## Sample Origin & Discovery

I discovered the suspicious file through a YouTube tutorial titled **"Best Free Hacked Client for Minecraft 1.8.9 | Infinite Scaffold, Killaura.... | myau client for free"**.

- **YouTube tutorial:** [Watch the video](https://www.youtube.com/watch?v=GcmhIIgANx4)
- **Sample download URL (MediaFire):** [Myau-250910+9.jar](https://www.mediafire.com/file/vmudh9t6w8kl5et/Myau-250910+9.jar/file)

> **Warning:** The MediaFire link is included solely for documentation and threat-research purposes. Do not execute the JAR on your primary device or with your personal Minecraft/Microsoft account. Its appearance in the linked tutorial does not, by itself, establish who created or modified the file.

---

## Technical Analysis

### 1. Anti-Analysis & Evasion (The Smoke Screen)

The mod contains a standard entry class `myau.init.Initializer` which merely prints a deceptive string to the console to simulate a clean startup:

```java
package myau.init;

public class Initializer {
   public Initializer() {
      System.out.println("Meow!");
   }
}
```

To prevent automated scanner tools (which parse static configuration files like `mixins.myau.json`) from detecting its custom injectors, the developer implemented a dynamic runtime class walker inside `myau.init.FMLLoadingPlugin`:

```java
public class FMLLoadingPlugin implements IMixinConfigPlugin {
   // ...
   public List<String> getMixins() {
      // Dynamically walks the jar file structure at runtime to inject hidden classes
      if (Files.isDirectory(file, new LinkOption[0])) {
         this.walkDir(file);
      } else {
         this.walkJar(file);
      }
      return this.mixins;
   }
}

```

This forces the Forge Mod Loader to register hidden Mixins inside `myau.mixin.*` without explicitly declaring them in the configuration manifest.

### 2. The Credential Harvesting Mechanism

The core payload is hidden inside a detached secondary package: `me.ksyz.accountmanager.auth.MicrosoftAuth`.

The execution routine triggers the following automated malicious sequence:

1. **Local Server Binding:** The malware initializes a hidden local HTTP daemon on a hardcoded port (`25575`) to listen for loopback authentication tokens:

   ```java
   HttpServer server = HttpServer.create(new InetSocketAddress(25575), 0);
   ```

2. **Victim Phishing Flow:** It forces the user's default system browser to navigate to the official Microsoft Live login portal, embedding the threat actor's rogue `client_id`:

   ```text
   https://login.live.com/oauth20_authorize.srf?client_id=[REDACTED]&redirect_uri=http://localhost:25575/callback
   ```

3. **Token Interception:** Once the victim logs in, Microsoft generates an authorization code and sends it to the loopback address. The mod intercepts this code using an embedded resource file `callback.html` which benignly instructs the user: *"Close this window and return to Minecraft!"*.

4. **Token Upgrading:** In the background, the code rapidly contacts Microsoft and Xbox Live API endpoints to upgrade the stolen login token into a full gaming session:

   * `https://login.live.com/oauth20_token.srf` (Exchanges auth code for access token)

   * `https://user.auth.xboxlive.com/user/authenticate` (Obtains UserToken)

   * `https://xsts.auth.xboxlive.com/xsts/authorize` (Obtains XSTS Token)

   * `https://api.minecraftservices.com/authentication/login_with_xbox` (Obtains Minecraft Profile Authorization)

### 3. Exfiltration

After resolving the profile through `https://api.minecraftservices.com/minecraft/profile`, the application collects the User UUID, Profile Name, and Session Token. The malicious actor relies on a local keystore asset embedded inside the archive (`ssl.jks`, ~176 KB) to establish a pinned, encrypted TLS connection directly to their Command & Control (C2) server, ensuring the exfiltrated credentials bypass network intrusion detection systems (IDS).

---

## Indicators of Compromise (IoCs)

### File Structural Red Flags

If you decompile this mod archive, look for these highly anomalous components:

* `callback.html` (~47 KB) - The HTML webflow interceptor page.

* `ssl.jks` (~176 KB) - Custom Java Keystore asset used for encrypted exfiltration.

* Presence of the `me.ksyz.accountmanager` package structure.

### Network Indicators

* Local listeners spawned on TCP port `25575`.

* Outbound browser triggers routing unexpectedly to `login.live.com` with local loopback redirects upon launch.

---

## Conclusion & Mitigation

This file is **100% malicious**. If you have downloaded `Myau-250910+9.jar` but **did not launch the game** with the mod active, your account is perfectly safe. The decompilation and static reading of the `.jar` archive do not trigger the payload.

### Remediation Steps (If you executed the mod):

1. **Revoke App Permissions:** Go to your Microsoft Account Security Dashboard -> Apps and Services, and revoke access to any unknown third-party apps or clients linked to Minecraft.

2. **Change Credentials:** Change your Microsoft account password immediately. This invalidates all active XSTS tokens harvested by the attacker.

3. **Purge the Malware:** Securely delete the `.jar` file and clear your client cache directories.
