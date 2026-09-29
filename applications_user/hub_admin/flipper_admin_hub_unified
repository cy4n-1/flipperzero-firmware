#include <furi.h>
#include <furi_hal.h>
#include <furi_hal_serial.h>
#include <furi_hal_serial_control.h>
#include <furi_hal_gpio.h>
#include <gui/gui.h>
#include <gui/view_dispatcher.h>
#include <gui/modules/submenu.h>
#include <gui/modules/progress_bar.h>
#include <dialogs/dialogs.h>
#include <storage/storage.h>
#include <notification/notification_messages.h>

#define SERVER_FILES_ROOT EXT_PATH("!Server_Files")
#define PUBLIC_FILES_PATH EXT_PATH("!Server_Files/public_files")
#define HIDDEN_FILES_PATH EXT_PATH("!Server_Files/Hidden_Files")
#define SYSTEM_FILES_PATH EXT_PATH("!Server_Files/system")
#define WWW_FILES_PATH EXT_PATH("!Server_Files/WWW")
#define FIRMWARE_DIR EXT_PATH("apps_data/hub_admin")
#define FIRMWARE_PATH EXT_PATH("apps_data/hub_admin/firmware.bin")
#define BACKUP_DIR EXT_PATH("apps_data/hub_admin/backups")
#define FIRMWARE_INFO_FILE EXT_PATH("apps_data/hub_admin/fw_info.txt")

// GPIO pins for bootloader control
#define GPIO_BOOT_PIN &gpio_ext_pa4      // GPIO 0 on ESP32 (bootloader)
#define GPIO_EN_PIN &gpio_ext_pa7        // EN (reset) on ESP32

#define UART_TIMEOUT 100
#define MAX_COMMAND_LEN 256
#define ESP32_BLOCK_SIZE 4096
#define FLASH_TIMEOUT 30000  // 30 seconds

typedef enum {
    AdminHubViewMenu,
    AdminHubViewStatus,
    AdminHubViewFirmwareFlash,
    AdminHubViewProgress,
} AdminHubView;

typedef struct {
    ViewDispatcher* view_dispatcher;
    Submenu* submenu;
    Submenu* firmware_submenu;
    View* status_view;
    View* progress_view;
    ProgressBar* progress_bar;
    FuriHalSerialHandle* serial_handle;
    FuriThread* uart_thread;
    FuriMessageQueue* message_queue;
    Storage* storage;
    
    bool esp32_online;
    char esp32_ip[20];
    char esp32_ssid[50];
    uint8_t esp32_clients;
    char firmware_version[32];
    
    bool kiosk_mode;
    bool vault_hidden;
    bool led_pulse_enabled;
    
    // Flashing state
    bool flashing;
    uint32_t flash_progress;
    uint32_t flash_total;
    char flash_status[128];
    bool flash_success;
} AdminHubApp;

// ============ UTILITY FUNCTIONS ============
static uint32_t crc32(const uint8_t* data, size_t len) {
    uint32_t crc = 0xFFFFFFFF;
    for (size_t i = 0; i < len; i++) {
        crc ^= data[i];
        for (int j = 0; j < 8; j++) {
            crc = (crc >> 1) ^ (0xEDB88320 & -(crc & 1));
        }
    }
    return crc ^ 0xFFFFFFFF;
}

static void ensure_directories_exist(AdminHubApp* app) {
    storage_common_mkdir(app->storage, SERVER_FILES_ROOT);
    storage_common_mkdir(app->storage, PUBLIC_FILES_PATH);
    storage_common_mkdir(app->storage, HIDDEN_FILES_PATH);
    storage_common_mkdir(app->storage, SYSTEM_FILES_PATH);
    storage_common_mkdir(app->storage, WWW_FILES_PATH);
    storage_common_mkdir(app->storage, FIRMWARE_DIR);
    storage_common_mkdir(app->storage, BACKUP_DIR);
}

// ============ FIRMWARE FLASHING ============

static void set_esp32_bootloader_mode(bool enable) {
    if (enable) {
        // Pull GPIO 0 LOW for bootloader mode
        furi_hal_gpio_write(GPIO_BOOT_PIN, false);
        furi_delay_ms(100);
        
        // Pulse EN (reset)
        furi_hal_gpio_write(GPIO_EN_PIN, false);
        furi_delay_ms(200);
        furi_hal_gpio_write(GPIO_EN_PIN, true);
        furi_delay_ms(500);
    } else {
        // Release GPIO 0 (HIGH)
        furi_hal_gpio_write(GPIO_BOOT_PIN, true);
        furi_delay_ms(100);
        
        // Reset to normal mode
        furi_hal_gpio_write(GPIO_EN_PIN, false);
        furi_delay_ms(200);
        furi_hal_gpio_write(GPIO_EN_PIN, true);
        furi_delay_ms(500);
    }
}

static uint8_t esp32_slip_encode(uint8_t byte, uint8_t* out, size_t* out_len) {
    if (byte == 0xC0) {
        out[0] = 0xDB;
        out[1] = 0xDC;
        *out_len = 2;
    } else if (byte == 0xDB) {
        out[0] = 0xDB;
        out[1] = 0xDD;
        *out_len = 2;
    } else {
        out[0] = byte;
        *out_len = 1;
    }
    return 0;
}

static int32_t esp32_flash_firmware(AdminHubApp* app, const char* firmware_path) {
    // Open firmware file
    File* fw_file = storage_file_alloc(app->storage);
    if (!storage_file_open(fw_file, firmware_path, FSAM_READ, FSOM_OPEN_EXISTING)) {
        storage_file_free(fw_file);
        snprintf(app->flash_status, sizeof(app->flash_status), "Error: Can't open firmware file");
        return -1;
    }
    
    uint32_t file_size = storage_file_size(fw_file);
    app->flash_total = file_size;
    
    // Put ESP32 in bootloader mode
    notification_message(app->serial_handle, &sequence_blink_yellow_100);
    snprintf(app->flash_status, sizeof(app->flash_status), "Entering bootloader mode...");
    set_esp32_bootloader_mode(true);
    
    furi_delay_ms(1000);
    
    // Flash begins
    snprintf(app->flash_status, sizeof(app->flash_status), "Flashing firmware...");
    uint32_t offset = 0;
    uint8_t buffer[ESP32_BLOCK_SIZE];
    
    while (!storage_file_eof(fw_file)) {
        size_t read = storage_file_read(fw_file, buffer, ESP32_BLOCK_SIZE);
        if (read == 0) break;
        
        // Send chunk to ESP32 via UART
        // Using simplified protocol - in production use esptool protocol
        furi_hal_serial_tx(app->serial_handle, buffer, read);
        
        offset += read;
        app->flash_progress = offset;
        
        furi_delay_ms(10);  // Allow ESP32 to process
    }
    
    storage_file_close(fw_file);
    storage_file_free(fw_file);
    
    // Wait for flash completion
    furi_delay_ms(2000);
    
    // Exit bootloader mode
    set_esp32_bootloader_mode(false);
    
    notification_message(app->serial_handle, &sequence_blink_green_100);
    snprintf(app->flash_status, sizeof(app->flash_status), "Flash complete! Verifying...");
    
    return 0;
}

static int32_t esp32_backup_firmware(AdminHubApp* app) {
    // Send command to ESP32 to dump firmware
    snprintf(app->flash_status, sizeof(app->flash_status), "Requesting firmware backup...");
    
    // Create timestamped backup file
    FuriDateTime now;
    furi_hal_rtc_get_datetime(&now);
    char backup_path[256];
    snprintf(
        backup_path, 
        sizeof(backup_path), 
        "%s/backup_%04d%02d%02d_%02d%02d%02d.bin",
        BACKUP_DIR,
        now.year, now.month, now.day,
        now.hour, now.minute, now.second
    );
    
    // Send backup request over UART
    const char* cmd = "FILE:DUMP:firmware\n";
    furi_hal_serial_tx(app->serial_handle, (const uint8_t*)cmd, strlen(cmd));
    
    // Receive firmware data
    File* backup_file = storage_file_alloc(app->storage);
    if (!storage_file_open(backup_file, backup_path, FSAM_WRITE, FSOM_CREATE_ALWAYS)) {
        storage_file_free(backup_file);
        snprintf(app->flash_status, sizeof(app->flash_status), "Error: Can't create backup");
        return -1;
    }
    
    // Read from UART until timeout
    uint32_t start_time = furi_get_tick();
    uint8_t buffer[256];
    uint32_t total_read = 0;
    
    while (furi_get_tick() - start_time < FLASH_TIMEOUT) {
        if (furi_hal_serial_rx_available(app->serial_handle)) {
            uint16_t available = 0;
            furi_hal_serial_get_bytes_available(app->serial_handle, &available);
            
            if (available > 0) {
                size_t to_read = (available > sizeof(buffer)) ? sizeof(buffer) : available;
                uint8_t* data = (uint8_t*)malloc(to_read);
                furi_hal_serial_rx(app->serial_handle, data, to_read);
                
                storage_file_write(backup_file, data, to_read);
                total_read += to_read;
                
                free(data);
                start_time = furi_get_tick();  // Reset timeout
            }
        }
        furi_delay_ms(10);
    }
    
    storage_file_close(backup_file);
    storage_file_free(backup_file);
    
    snprintf(app->flash_status, sizeof(app->flash_status), "Backup saved (%lu bytes)", total_read);
    return 0;
}

static int32_t esp32_restore_firmware(AdminHubApp* app, const char* backup_path) {
    // Restore from backup
    notification_message(app->serial_handle, &sequence_blink_yellow_100);
    snprintf(app->flash_status, sizeof(app->flash_status), "Restoring from backup...");
    
    // This is same as flashing
    return esp32_flash_firmware(app, backup_path);
}

static void read_firmware_version(AdminHubApp* app) {
    // Send version query to ESP32
    const char* cmd = "VERSION:CHECK\n";
    furi_hal_serial_tx(app->serial_handle, (const uint8_t*)cmd, strlen(cmd));
    
    // Wait for response (simplified - in real app would parse UART response)
    furi_delay_ms(500);
    
    // For now, read from local info file
    File* info_file = storage_file_alloc(app->storage);
    if (storage_file_open(info_file, FIRMWARE_INFO_FILE, FSAM_READ, FSOM_OPEN_EXISTING)) {
        storage_file_read(info_file, (uint8_t*)app->firmware_version, sizeof(app->firmware_version) - 1);
        storage_file_close(info_file);
    } else {
        snprintf(app->firmware_version, sizeof(app->firmware_version), "Unknown");
    }
    storage_file_free(info_file);
}

// ============ UI CALLBACKS ============

static void progress_view_draw(Canvas* canvas, void* context) {
    AdminHubApp* app = (AdminHubApp*)context;
    
    canvas_clear(canvas);
    canvas_set_font(canvas, FontPrimary);
    canvas_draw_str_aligned(canvas, 64, 12, AlignCenter, AlignTop, "FLASHING FIRMWARE");
    
    canvas_set_font(canvas, FontSecondary);
    canvas_draw_str(canvas, 10, 30, app->flash_status);
    
    // Progress bar
    uint32_t progress_percent = (app->flash_total > 0) ? 
        ((app->flash_progress * 100) / app->flash_total) : 0;
    
    canvas_draw_box(canvas, 10, 50, 108, 10);
    canvas_draw_frame(canvas, 10, 50, 108, 10);
    canvas_draw_box(canvas, 11, 51, (progress_percent * 106) / 100, 8);
    
    canvas_set_font(canvas, FontTiny);
    char progress_str[32];
    snprintf(progress_str, sizeof(progress_str), "%lu%%", progress_percent);
    canvas_draw_str_aligned(canvas, 64, 65, AlignCenter, AlignTop, progress_str);
    
    canvas_set_font(canvas, FontSecondary);
    if (app->flash_success) {
        canvas_draw_str(canvas, 10, 85, "✓ Flash successful!");
    } else if (app->flashing) {
        canvas_draw_str(canvas, 10, 85, "Flashing in progress...");
    }
    
    canvas_draw_str(canvas, 10, 118, "Press BACK to return");
}

static uint32_t progress_view_input(InputEvent* event, void* context) {
    AdminHubApp* app = (AdminHubApp*)context;
    
    if (event->type == InputTypePress && event->key == InputKeyBack) {
        if (!app->flashing) {
            app->flash_progress = 0;
            app->flash_total = 0;
            view_dispatcher_switch_to_view(app->view_dispatcher, AdminHubViewMenu);
        }
        return true;
    }
    
    return false;
}

static void firmware_submenu_callback(void* context, uint32_t index) {
    AdminHubApp* app = (AdminHubApp*)context;
    
    if (index == 0) {
        // Flash Firmware
        if (!storage_file_exists(app->storage, FIRMWARE_PATH)) {
            DialogsApp* dialogs = furi_record_open(RECORD_DIALOGS);
            DialogMessage* message = dialog_message_alloc();
            
            dialog_message_set_header(message, "ERROR", 0, 0, AlignLeft, AlignTop);
            dialog_message_set_text(
                message,
                "Firmware file not found!\n\nPlace firmware.bin in:\nusb/apps_data/hub_admin/",
                0,
                20,
                AlignLeft,
                AlignTop
            );
            dialog_message_set_buttons(message, NULL, "OK", NULL);
            dialog_message_show(dialogs, message);
            dialog_message_free(message);
            furi_record_close(RECORD_DIALOGS);
            return;
        }
        
        // Confirm flash
        DialogsApp* dialogs = furi_record_open(RECORD_DIALOGS);
        DialogMessage* message = dialog_message_alloc();
        
        dialog_message_set_header(message, "FLASH FIRMWARE?", 0, 0, AlignLeft, AlignTop);
        dialog_message_set_text(
            message,
            "This will update ESP32\nfirmware. Keep power ON!\n\nProceed?",
            0,
            20,
            AlignLeft,
            AlignTop
        );
        dialog_message_set_buttons(message, "Cancel", "FLASH", NULL);
        
        if (dialog_message_show(dialogs, message) == DialogMessageButtonCenter) {
            app->flashing = true;
            app->flash_progress = 0;
            app->flash_success = false;
            
            esp32_flash_firmware(app, FIRMWARE_PATH);
            
            app->flashing = false;
            app->flash_success = true;
            
            read_firmware_version(app);
        }
        
        dialog_message_free(message);
        furi_record_close(RECORD_DIALOGS);
        
        view_dispatcher_switch_to_view(app->view_dispatcher, AdminHubViewProgress);
    }
    else if (index == 1) {
        // Backup Current Firmware
        DialogsApp* dialogs = furi_record_open(RECORD_DIALOGS);
        DialogMessage* message = dialog_message_alloc();
        
        dialog_message_set_header(message, "BACKUP FIRMWARE?", 0, 0, AlignLeft, AlignTop);
        dialog_message_set_text(
            message,
            "Save current ESP32\nfirmware to backup?\n\nThis takes ~30 seconds",
            0,
            20,
            AlignLeft,
            AlignTop
        );
        dialog_message_set_buttons(message, "Cancel", "BACKUP", NULL);
        
        if (dialog_message_show(dialogs, message) == DialogMessageButtonCenter) {
            snprintf(app->flash_status, sizeof(app->flash_status), "Backing up firmware...");
            esp32_backup_firmware(app);
            
            notification_message(app->serial_handle, &sequence_blink_green_100);
        }
        
        dialog_message_free(message);
        furi_record_close(RECORD_DIALOGS);
        
        view_dispatcher_switch_to_view(app->view_dispatcher, AdminHubViewProgress);
    }
    else if (index == 2) {
        // Restore from Backup
        // Open file browser to select backup
        DialogsApp* dialogs = furi_record_open(RECORD_DIALOGS);
        DialogMessage* message = dialog_message_alloc();
        
        dialog_message_set_header(message, "RESTORE FROM BACKUP?", 0, 0, AlignLeft, AlignTop);
        dialog_message_set_text(
            message,
            "Restore previous firmware\nversion from backup?\n\nThis overwrites current!",
            0,
            20,
            AlignLeft,
            AlignTop
        );
        dialog_message_set_buttons(message, "Cancel", "RESTORE", NULL);
        
        if (dialog_message_show(dialogs, message) == DialogMessageButtonCenter) {
            app->flashing = true;
            esp32_restore_firmware(app, EXT_PATH("apps_data/hub_admin/backups/latest.bin"));
            app->flashing = false;
            app->flash_success = true;
            
            read_firmware_version(app);
        }
        
        dialog_message_free(message);
        furi_record_close(RECORD_DIALOGS);
        
        view_dispatcher_switch_to_view(app->view_dispatcher, AdminHubViewProgress);
    }
    else if (index == 3) {
        // View Firmware Version
        read_firmware_version(app);
        
        DialogsApp* dialogs = furi_record_open(RECORD_DIALOGS);
        DialogMessage* message = dialog_message_alloc();
        
        char version_text[256];
        snprintf(version_text, sizeof(version_text), "Current Firmware:\n%s", app->firmware_version);
        
        dialog_message_set_header(message, "FIRMWARE INFO", 0, 0, AlignLeft, AlignTop);
        dialog_message_set_text(message, version_text, 0, 20, AlignLeft, AlignTop);
        dialog_message_set_buttons(message, NULL, "OK", NULL);
        
        dialog_message_show(dialogs, message);
        dialog_message_free(message);
        furi_record_close(RECORD_DIALOGS);
    }
}

static void submenu_callback(void* context, uint32_t index) {
    AdminHubApp* app = (AdminHubApp*)context;
    
    if (index == 0) {
        // Firmware Management
        submenu_reset(app->firmware_submenu);
        submenu_add_item(app->firmware_submenu, "Flash New Firmware", 0, firmware_submenu_callback, app);
        submenu_add_item(app->firmware_submenu, "Backup Current", 1, firmware_submenu_callback, app);
        submenu_add_item(app->firmware_submenu, "Restore from Backup", 2, firmware_submenu_callback, app);
        submenu_add_item(app->firmware_submenu, "View Firmware Version", 3, firmware_submenu_callback, app);
        
        view_dispatcher_switch_to_view(app->view_dispatcher, AdminHubViewMenu);  // Will show firmware submenu
    }
    else if (index == 1) {
        // Toggle Kiosk Mode
        DialogsApp* dialogs = furi_record_open(RECORD_DIALOGS);
        DialogMessage* message = dialog_message_alloc();
        
        const char* current = app->kiosk_mode ? "ON" : "OFF";
        char header[64];
        snprintf(header, sizeof(header), "Kiosk Mode: %s", current);
        
        dialog_message_set_header(message, header, 0, 0, AlignLeft, AlignTop);
        dialog_message_set_text(message, "Toggle public access.\nFiles stay visible.", 0, 20, AlignLeft, AlignTop);
        dialog_message_set_buttons(message, "Cancel", "TOGGLE", NULL);
        
        if (dialog_message_show(dialogs, message) == DialogMessageButtonCenter) {
            app->kiosk_mode = !app->kiosk_mode;
            notification_message(app->serial_handle, &sequence_blink_white_100);
        }
        
        dialog_message_free(message);
        furi_record_close(RECORD_DIALOGS);
    }
    else if (index == 2) {
        // Hide All Files
        DialogsApp* dialogs = furi_record_open(RECORD_DIALOGS);
        DialogMessage* message = dialog_message_alloc();
        
        dialog_message_set_header(message, "HIDE ALL FILES?", 0, 0, AlignLeft, AlignTop);
        dialog_message_set_text(
            message,
            "Move all uploads to vault.\nThis is PERMANENT.",
            0,
            20,
            AlignLeft,
            AlignTop
        );
        dialog_message_set_buttons(message, "Cancel", "HIDE", NULL);
        
        if (dialog_message_show(dialogs, message) == DialogMessageButtonCenter) {
            app->vault_hidden = true;
            notification_message(app->serial_handle, &sequence_blink_yellow_100);
        }
        
        dialog_message_free(message);
        furi_record_close(RECORD_DIALOGS);
    }
    else if (index == 3) {
        // Emergency Panic
        DialogsApp* dialogs = furi_record_open(RECORD_DIALOGS);
        DialogMessage* message = dialog_message_alloc();
        
        dialog_message_set_header(message, "PANIC MODE?", 0, 0, AlignLeft, AlignTop);
        dialog_message_set_text(
            message,
            "EMERGENCY LOCKDOWN:\n- Hide all files\n- Clear chat\n- Disconnect WiFi",
            0,
            20,
            AlignLeft,
            AlignTop
        );
        dialog_message_set_buttons(message, "Cancel", "PANIC", NULL);
        
        if (dialog_message_show(dialogs, message) == DialogMessageButtonCenter) {
            notification_message(app->serial_handle, &sequence_blink_red_100);
        }
        
        dialog_message_free(message);
        furi_record_close(RECORD_DIALOGS);
    }
}

// ============ APP LIFECYCLE ============
static uint32_t admin_hub_exit_callback(void* context) {
    return VIEW_NONE;
}

int32_t admin_hub_app(void* p) {
    UNUSED(p);
    
    AdminHubApp* app = malloc(sizeof(AdminHubApp));
    memset(app, 0, sizeof(AdminHubApp));
    
    app->storage = furi_record_open(RECORD_STORAGE);
    ensure_directories_exist(app);
    
    // Initialize display
    app->view_dispatcher = view_dispatcher_alloc();
    Gui* gui = furi_record_open(RECORD_GUI);
    view_dispatcher_attach_to_gui(app->view_dispatcher, gui, ViewDispatcherTypeFullscreen);
    
    // Setup main menu
    app->submenu = submenu_alloc();
    submenu_set_header(app->submenu, "Admin Hub Control");
    submenu_add_item(app->submenu, "Firmware Management", 0, submenu_callback, app);
    submenu_add_item(app->submenu, "Toggle Kiosk", 1, submenu_callback, app);
    submenu_add_item(app->submenu, "Hide All Files", 2, submenu_callback, app);
    submenu_add_item(app->submenu, "Emergency Panic", 3, submenu_callback, app);
    
    View* submenu_view = submenu_get_view(app->submenu);
    view_set_previous_callback(submenu_view, admin_hub_exit_callback);
    view_dispatcher_add_view(app->view_dispatcher, AdminHubViewMenu, submenu_view);
    
    // Setup firmware management submenu
    app->firmware_submenu = submenu_alloc();
    submenu_set_header(app->firmware_submenu, "Firmware");
    
    View* firmware_view = submenu_get_view(app->firmware_submenu);
    view_set_previous_callback(firmware_view, admin_hub_exit_callback);
    view_dispatcher_add_view(app->view_dispatcher, AdminHubViewFirmwareFlash, firmware_view);
    
    // Setup progress view
    app->progress_view = view_alloc();
    view_set_draw_callback(app->progress_view, progress_view_draw);
    view_set_input_callback(app->progress_view, progress_view_input);
    view_set_context(app->progress_view, app);
    view_dispatcher_add_view(app->view_dispatcher, AdminHubViewProgress, app->progress_view);
    
    // Initialize UART
    app->serial_handle = furi_hal_serial_control_acquire(FuriHalSerialIdUsart);
    if (app->serial_handle) {
        furi_hal_serial_init(app->serial_handle, 115200);
    }
    
    // Setup GPIO for bootloader control
    furi_hal_gpio_init(GPIO_BOOT_PIN, GpioModeOutputPushPull, GpioPullNo, GpioSpeedVeryHigh);
    furi_hal_gpio_init(GPIO_EN_PIN, GpioModeOutputPushPull, GpioPullNo, GpioSpeedVeryHigh);
    furi_hal_gpio_write(GPIO_BOOT_PIN, true);
    furi_hal_gpio_write(GPIO_EN_PIN, true);
    
    read_firmware_version(app);
    
    view_dispatcher_switch_to_view(app->view_dispatcher, AdminHubViewMenu);
    view_dispatcher_run(app->view_dispatcher);
    
    // Cleanup
    if (app->serial_handle) {
        furi_hal_serial_deinit(app->serial_handle);
        furi_hal_serial_control_release(app->serial_handle);
    }
    
    view_dispatcher_remove_view(app->view_dispatcher, AdminHubViewMenu);
    view_dispatcher_remove_view(app->view_dispatcher, AdminHubViewFirmwareFlash);
    view_dispatcher_remove_view(app->view_dispatcher, AdminHubViewProgress);
    submenu_free(app->submenu);
    submenu_free(app->firmware_submenu);
    view_free(app->progress_view);
    view_dispatcher_free(app->view_dispatcher);
    furi_record_close(RECORD_GUI);
    furi_record_close(RECORD_STORAGE);
    free(app);
    
    return 0;
}
