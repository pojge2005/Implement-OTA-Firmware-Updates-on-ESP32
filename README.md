# Implement-OTA-Firmware-Updates-on-ESP32

ESP - You’ve deployed 50 ESP32 devices across a building. Now you’ve found a bug. The math is brutal: 50 devices × 5 minutes each × the cost of your time = a very expensive afternoon with a USB cable.

This is the moment every ESP32 developer hits, when flashing via serial stops being convenient and starts being a liability. OTA (over-the-air) firmware updates solve this by letting you push new firmware to your devices over WiFi, whether they’re across the room or across the country.

By the end of this guide, you’ll have a working OTA implementation using ESP-IDF’s native esp_https_ota component. We’ll cover the partition scheme that makes safe updates possible, write the client code that fetches and applies firmware, and address the security considerations you can’t ignore in production.
Prerequisites: A working ESP-IDF environment (v5.0+), basic WiFi connectivity knowledge, and an ESP32 with at least 4MB flash.
How the Dual-Partition Scheme Keeps Updates Safe 
Here’s the problem OTA has to solve: you can’t update the firmware you’re currently running. It’s like trying to change a car’s engine while driving.

ESP32 solves this with a dual-partition scheme. Your flash contains two application partitions, ota_0 and ota_1, and a small otadata partition that tracks which one is active.

┌─────────────────┐
│    Bootloader   │
├─────────────────┤
│  Partition Table│
├─────────────────┤
│     otadata     │  ← Tracks active partition
├─────────────────┤
│     ota_0       │  ← App partition A
├─────────────────┤
│     ota_1       │  ← App partition B


When you perform an OTA update:

-New firmware writes to the inactive partition
-The otadata partition updates to mark the new partition as “pending verification”
-The device reboots into the new firmware
-If the new firmware runs successfully and marks itself valid, it becomes the active partition
-If it crashes, the bootloader automatically rolls back to the previous working version

This design means a failed update never bricks your device. The previous firmware remains intact and ready to take over.
Configuring Your Project for OTA 
Before writing code, your project needs the correct partition table. The default “Single factory app, no OTA” configuration won’t work.

Open menuconfig:

idf.py menuconfig

Navigate to Partition Table and select Factory app, two OTA definitions. This gives you:

A factory partition (your initial firmware)
Two OTA partitions for updates
The otadata partition for boot management
For a 4MB flash, each OTA partition will be roughly 1.5MB, which is plenty for most applications. If your application is larger, you’ll need to customize the partition table or use 8MB+ flash.

No additional component configuration is needed; esp_https_ota is included in ESP-IDF by default.
Implementing the OTA Client 
The core OTA implementation is surprisingly compact. ESP-IDF’s esp_https_ota() function handles the heavy lifting: connecting to your server, streaming the firmware in chunks, writing to flash, and verifying the image.

Here’s a complete OTA update function:  


#include "esp_https_ota.h"
#include "esp_log.h"
#include "esp_ota_ops.h"

static const char *TAG = "OTA";

// For production, embed your server's CA certificate
extern const uint8_t server_cert_pem_start[] asm("_binary_ca_cert_pem_start");
extern const uint8_t server_cert_pem_end[] asm("_binary_ca_cert_pem_end");

esp_err_t perform_ota_update(const char *firmware_url)
{
    ESP_LOGI(TAG, "Starting OTA update from %s", firmware_url);

    esp_http_client_config_t http_config = {
        .url = firmware_url,
        .cert_pem = (const char *)server_cert_pem_start,
        .timeout_ms = 30000,
        .keep_alive_enable = true,
    };

    esp_https_ota_config_t ota_config = {
        .http_config = &http_config,
    };

    esp_err_t ret = esp_https_ota(&ota_config);
    
    if (ret == ESP_OK) {
        ESP_LOGI(TAG, "OTA update successful. Rebooting...");
        esp_restart();
    } else {
        ESP_LOGE(TAG, "OTA update failed: %s", esp_err_to_name(ret));
    }
    
    return ret;
}


A few things to note:

The certificate embedding uses ESP-IDF’s binary embedding feature. Add your CA certificate to your project and reference it in CMakeLists.txt:

target_add_binary_data(${COMPONENT_TARGET} "ca_cert.pem" TEXT)

The function blocks until the update completes or fails. For production, you might want the non-blocking esp_https_ota_begin()/esp_https_ota_perform()/esp_https_ota_finish() sequence to show progress or handle cancellation.

Triggering the update is your design choice. Common approaches:

Check a version endpoint on boot or periodically
Listen for an MQTT message commanding an update
Respond to a button press or API call
Poll a server at regular intervals
Hosting Your Firmware Binary 
Your OTA client needs somewhere to fetch firmware from. During development, a local HTTPS server works fine. For local testing, Python’s HTTP server gets you started quickly:

cd build
python3 -m http.server 8070

Your firmware URL becomes http://192.168.1.x:8070/your_project.bin. Note: this is HTTP, not HTTPS, which is acceptable only for isolated development networks.

For production, you need HTTPS. Options include:

AWS S3 + CloudFront: Reliable, scalable, pay-per-use
GitHub Releases: Free for open-source projects
Your own server: nginx with Let’s Encrypt certificates 
The firmware binary is always at build/your_project_name.bin after running idf.py build.

Adding Version Checking 
Without version checking, your devices will re-download and re-flash identical firmware every time they check for updates. This wastes bandwidth, wears flash memory, and risks update failures during the unnecessary writes.

ESP-IDF embeds version information in every build. Access it with: 
#include "esp_app_desc.h"

bool should_update(const char *new_version)
{
    const esp_app_desc_t *app_desc = esp_app_get_description();
    
    ESP_LOGI(TAG, "Current version: %s", app_desc->version);
    ESP_LOGI(TAG, "Available version: %s", new_version);
    
    // Simple string comparison; customize for semantic versioning
    return strcmp(app_desc->version, new_version) != 0;
} 
Set your version in CMakeLists.txt:

set(PROJECT_VER "1.0.1")

For the server side, options include:

A separate /version.txt endpoint your device checks first
HTTP headers in the firmware response
A manifest JSON file with version and URL
The simplest approach: fetch the version string before initiating OTA, compare, and only proceed if different.
Security You Can’t Skip in Production 
OTA is a powerful capability and a significant attack vector. A compromised update mechanism means an attacker can flash arbitrary code to your devices.

HTTPS is non-negotiable. The certificate verification in the code above prevents man-in-the-middle attacks. For development, you might be tempted to disable verification:

.skip_cert_common_name_check = true,  // NEVER in production

Don’t ship this. Ever. An attacker on the same network could intercept the update request and serve malicious firmware.

For production deployments, also consider: 
Secure Boot: Ensures only signed bootloaders and apps run
Flash Encryption: Protects firmware from physical extraction
Signed App Images: Verifies firmware authenticity before flashing
These are substantial topics deserving dedicated coverage. At minimum, ship with HTTPS verification enabled and certificates properly pinned.

Testing Your Implementation 
A methodical first test saves debugging headaches:


const esp_partition_t *running = esp_ota_get_running_partition();
ESP_LOGI(TAG, "Running from partition: %s", running->label);

Test rollback by intentionally flashing firmware that crashes early (before calling esp_ota_mark_app_valid_cancel_rollback()). After a few failed boots, the device should revert to the previous working version.

Common issues:

WiFi disconnects during download: Increase timeouts, ensure stable connection before starting
Timeout errors: Large firmware over slow connections needs longer timeout_ms
Partition too small: Check your binary size against partition table allocations
Integrating OTA Into Your Project 
You now have the building blocks for production-ready OTA updates. The immediate next steps:
Add the OTA function to your existing project
Set up a proper HTTPS firmware hosting location
Implement version checking to prevent redundant updates
Define how updates are triggered (periodic check, command, manual)
For production deployments, you’ll eventually want to explore staged rollouts (update 10% of devices, verify, then expand), automated rollback policies, and delta updates for bandwidth-constrained scenarios. But the foundation you’ve built here, with dual partitions, secure downloads, and version checking, will serve you from prototypes through production.

The USB cable isn’t going anywhere. You’ll still need it for initial provisioning and recovery. But for the day-to-day reality of maintaining deployed devices, OTA transforms firmware updates from a logistics problem into a software problem. And software problems, unlike driving across town with a laptop, scale well.





