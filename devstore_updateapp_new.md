# Full Implementation: Custom App Store & In-App Update System

This document contains the complete source code, XML layouts, and manifest configurations for a professional in-app update system and a developer app store.

---

## 1. Android Manifest Configuration (`AndroidManifest.xml`)

Add these permissions, queries, and activity declarations to your manifest.

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="YOUR_PACKAGE_NAME">

    <!-- Permissions -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES" />
    <uses-permission android:name="android.permission.DOWNLOAD_WITHOUT_NOTIFICATION" />
    <!-- Needed for installation on API 28 and below -->
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" android:maxSdkVersion="28" />

    <!-- Queries for detecting if apps are installed -->
    <queries>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </queries>

    <application ...>

        <!-- Developer App Store Activities -->
        <activity
            android:name=".ui.developerstore.DeveloperStoreActivity"
            android:screenOrientation="portrait" />

        <activity
            android:name=".ui.developerstore.AppDetailsActivity"
            android:screenOrientation="portrait" />

        <!-- Self Update Download Activity -->
        <activity
            android:name=".ui.update.UpdateDownloadActivity"
            android:exported="false"
            android:screenOrientation="portrait" />

        <!-- File Provider for APK Installation -->
        <provider
            android:name="androidx.core.content.FileProvider"
            android:authorities="${applicationId}.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_provider_paths" />
        </provider>

    </application>
</manifest>
```

---

## 2. File Provider Paths (`res/xml/file_provider_paths.xml`)

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <!-- Maps to /Android/data/your.package.name/files/ -->
    <external-path name="external_files" path="." />
</paths>
```

---

## 3. Data Model (`AppStoreItem.java`)

Used for both the App Store list and the Update info.

```java
package YOUR_PACKAGE_NAME.data.model;

public class AppStoreItem {
    private String title;
    private String iconUrl;
    private String downloadUrl;
    private String description;
    private String packageName;
    private String versionName;
    private int versionCode;

    public AppStoreItem() {}

    public AppStoreItem(String title, String iconUrl, String downloadUrl, String description, String packageName, String versionName, int versionCode) {
        this.title = title;
        this.iconUrl = iconUrl;
        this.downloadUrl = downloadUrl;
        this.description = description;
        this.packageName = packageName;
        this.versionName = versionName;
        this.versionCode = versionCode;
    }

    // Getters and Setters
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    public String getIconUrl() { return iconUrl; }
    public void setIconUrl(String iconUrl) { this.iconUrl = iconUrl; }
    public String getDownloadUrl() { return downloadUrl; }
    public void setDownloadUrl(String downloadUrl) { this.downloadUrl = downloadUrl; }
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
    public String getPackageName() { return packageName; }
    public void setPackageName(String packageName) { this.packageName = packageName; }
    public String getVersionName() { return versionName; }
    public void setVersionName(String versionName) { this.versionName = versionName; }
    public int getVersionCode() { return versionCode; }
    public void setVersionCode(int versionCode) { this.versionCode = versionCode; }

    public String getAvatarUrl() {
        if (iconUrl != null && !iconUrl.trim().isEmpty()) return iconUrl;
        String name = title != null ? title.replace(" ", "+") : "App";
        return "https://ui-avatars.com/api/?background=00BFA6&color=ffffff&bold=true&size=256&name=" + name;
    }
}
```

---

## 4. Core Logic Utilities

### `UpdateChecker.java`

```java
package YOUR_PACKAGE_NAME.core.utils;

import android.content.Context;
import android.os.Build;
import android.os.Handler;
import android.os.Looper;
import org.json.JSONObject;
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.net.HttpURLConnection;
import java.net.URL;
import YOUR_PACKAGE_NAME.data.model.AppStoreItem;

public class UpdateChecker {

    public interface UpdateCheckCallback {
        void onUpdateAvailable(AppStoreItem info);
        void onUpToDate();
        void onError(Exception e);
    }

    public static void check(Context context, String jsonUrl, UpdateCheckCallback callback) {
        new Thread(() -> {
            try {
                URL url = new URL(jsonUrl);
                HttpURLConnection connection = (HttpURLConnection) url.openConnection();
                connection.setRequestMethod("GET");
                connection.setConnectTimeout(10000);

                BufferedReader reader = new BufferedReader(new InputStreamReader(connection.getInputStream()));
                StringBuilder response = new StringBuilder();
                String line;
                while ((line = reader.readLine()) != null) response.append(line);
                reader.close();

                JSONObject json = new JSONObject(response.toString());
                AppStoreItem info = new AppStoreItem();
                info.setVersionCode(json.getInt("versionCode"));
                info.setVersionName(json.getString("versionName"));
                info.setDownloadUrl(json.getString("apkUrl"));

                int currentVersionCode = getInstalledVersionCode(context, context.getPackageName());

                new Handler(Looper.getMainLooper()).post(() -> {
                    if (info.getVersionCode() > currentVersionCode) {
                        callback.onUpdateAvailable(info);
                    } else {
                        callback.onUpToDate();
                    }
                });
            } catch (Exception e) {
                new Handler(Looper.getMainLooper()).post(() -> callback.onError(e));
            }
        }).start();
    }

    public static int getInstalledVersionCode(Context context, String packageName) {
        if (packageName == null || packageName.isEmpty()) return -1;
        try {
            android.content.pm.PackageInfo pInfo = context.getPackageManager().getPackageInfo(packageName, 0);
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
                return (int) pInfo.getLongVersionCode();
            } else {
                return pInfo.versionCode;
            }
        } catch (android.content.pm.PackageManager.NameNotFoundException e) {
            return -1;
        }
    }
}
```

---

## 5. UI Implementation

### `UpdateDownloadActivity.java`

```java
package YOUR_PACKAGE_NAME.ui.update;

import android.content.Intent;
import android.net.Uri;
import android.os.Bundle;
import android.os.Handler;
import android.os.Looper;
import android.widget.Toast;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.content.FileProvider;
import java.io.File;
import java.io.FileOutputStream;
import java.io.InputStream;
import java.net.HttpURLConnection;
import java.net.URL;
import YOUR_PACKAGE_NAME.databinding.ActivityUpdateDownloadBinding;

public class UpdateDownloadActivity extends AppCompatActivity {

    private ActivityUpdateDownloadBinding binding;
    private File outputFile;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        binding = ActivityUpdateDownloadBinding.inflate(getLayoutInflater());
        setContentView(binding.getRoot());

        String apkUrl = getIntent().getStringExtra("apk_url");
        String versionName = getIntent().getStringExtra("version_name");
        String appTitle = getIntent().getStringExtra("app_title");

        if (appTitle != null) binding.tvUpdateTitle.setText("Downloading " + appTitle);
        binding.tvUpdateVersion.setText("Version: " + versionName);

        binding.btnInstallUpdate.setOnClickListener(v -> installApk());

        if (apkUrl != null) downloadApk(apkUrl);
        else finish();
    }

    private void downloadApk(String urlString) {
        new Thread(() -> {
            try {
                URL url = new URL(urlString);
                HttpURLConnection conn = (HttpURLConnection) url.openConnection();
                int fileLength = conn.getContentLength();
                outputFile = new File(getExternalFilesDir(null), "update.apk");

                InputStream input = conn.getInputStream();
                FileOutputStream output = new FileOutputStream(outputFile);

                byte[] data = new byte[4096];
                long total = 0;
                int count;
                while ((count = input.read(data)) != -1) {
                    total += count;
                    int progress = (fileLength > 0) ? (int) (total * 100 / fileLength) : 0;
                    long finalTotal = total;
                    new Handler(Looper.getMainLooper()).post(() -> {
                        binding.progressDownload.setProgress(progress);
                        binding.tvProgressDetails.setText(progress + "% (" + (finalTotal/1024/1024) + "MB / " + (fileLength/1024/1024) + "MB)");
                    });
                    output.write(data, 0, count);
                }
                output.close(); input.close();
                new Handler(Looper.getMainLooper()).post(() -> {
                    binding.btnInstallUpdate.setEnabled(true);
                    installApk();
                });
            } catch (Exception e) {
                new Handler(Looper.getMainLooper()).post(() -> finish());
            }
        }).start();
    }

    private void installApk() {
        if (outputFile != null && outputFile.exists()) {
            Uri uri = FileProvider.getUriForFile(this, getPackageName() + ".fileprovider", outputFile);
            Intent intent = new Intent(Intent.ACTION_VIEW);
            intent.setDataAndType(uri, "application/vnd.android.package-archive");
            intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION | Intent.FLAG_ACTIVITY_NEW_TASK);
            startActivity(intent);
        }
    }
}
```

### `DeveloperStoreActivity.java`

```java
package YOUR_PACKAGE_NAME.ui.developerstore;

import android.content.Intent;
import android.net.Uri;
import android.os.Bundle;
import android.os.Handler;
import android.os.Looper;
import android.view.View;
import androidx.appcompat.app.AppCompatActivity;
import androidx.recyclerview.widget.LinearLayoutManager;
import org.json.JSONArray;
import org.json.JSONObject;
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.net.HttpURLConnection;
import java.net.URL;
import java.util.ArrayList;
import java.util.List;
import YOUR_PACKAGE_NAME.data.model.AppStoreItem;
import YOUR_PACKAGE_NAME.databinding.ActivityDeveloperStoreBinding;

public class DeveloperStoreActivity extends AppCompatActivity implements DeveloperStoreAdapter.OnAppClickListener {

    private ActivityDeveloperStoreBinding binding;
    private DeveloperStoreAdapter adapter;
    private static final String APPS_JSON_URL = "https://raw.githubusercontent.com/kirubanandem/appstore_data/refs/heads/main/apps.json";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        binding = ActivityDeveloperStoreBinding.inflate(getLayoutInflater());
        setContentView(binding.getRoot());
        adapter = new DeveloperStoreAdapter(this);
        binding.rvApps.setLayoutManager(new LinearLayoutManager(this));
        binding.rvApps.setAdapter(adapter);
        loadApps();
    }

    @Override
    protected void onResume() {
        super.onResume();
        if (adapter != null) adapter.notifyDataSetChanged();
    }

    private void loadApps() {
        binding.progressBar.setVisibility(View.VISIBLE);
        new Thread(() -> {
            try {
                URL url = new URL(APPS_JSON_URL);
                HttpURLConnection conn = (HttpURLConnection) url.openConnection();
                BufferedReader reader = new BufferedReader(new InputStreamReader(conn.getInputStream()));
                StringBuilder res = new StringBuilder();
                String line;
                while ((line = reader.readLine()) != null) res.append(line);
                reader.close();

                JSONArray arr = new JSONArray(res.toString());
                List<AppStoreItem> apps = new ArrayList<>();
                for (int i = 0; i < arr.length(); i++) {
                    JSONObject obj = arr.getJSONObject(i);
                    apps.add(new AppStoreItem(obj.getString("title"), obj.getString("iconUrl"), obj.getString("downloadUrl"), 
                        obj.getString("description"), obj.optString("packageName", ""), obj.optString("versionName", ""), obj.optInt("versionCode", 0)));
                }
                new Handler(Looper.getMainLooper()).post(() -> {
                    binding.progressBar.setVisibility(View.GONE);
                    adapter.setItems(apps);
                });
            } catch (Exception e) {
                new Handler(Looper.getMainLooper()).post(() -> binding.progressBar.setVisibility(View.GONE));
            }
        }).start();
    }

    @Override
    public void onDetailsClick(AppStoreItem item) {
        Intent intent = new Intent(this, AppDetailsActivity.class);
        intent.putExtra(AppDetailsActivity.EXTRA_TITLE, item.getTitle());
        intent.putExtra(AppDetailsActivity.EXTRA_DESC, item.getDescription());
        intent.putExtra(AppDetailsActivity.EXTRA_ICON, item.getIconUrl());
        intent.putExtra(AppDetailsActivity.EXTRA_URL, item.getDownloadUrl());
        intent.putExtra(AppDetailsActivity.EXTRA_PKG, item.getPackageName());
        intent.putExtra(AppDetailsActivity.EXTRA_VERSION_NAME, item.getVersionName());
        intent.putExtra(AppDetailsActivity.EXTRA_VERSION_CODE, item.getVersionCode());
        startActivity(intent);
    }
}
```

### `AppDetailsActivity.java`

```java
package YOUR_PACKAGE_NAME.ui.developerstore;

import android.content.Intent;
import android.net.Uri;
import android.os.Bundle;
import android.view.View;
import androidx.appcompat.app.AppCompatActivity;
import com.bumptech.glide.Glide;
import YOUR_PACKAGE_NAME.R;
import YOUR_PACKAGE_NAME.databinding.ActivityAppDetailsBinding;
import YOUR_PACKAGE_NAME.core.utils.UpdateChecker;

public class AppDetailsActivity extends AppCompatActivity {

    public static final String EXTRA_TITLE = "extra_title", EXTRA_DESC = "extra_desc", EXTRA_ICON = "extra_icon", 
                               EXTRA_URL = "extra_url", EXTRA_PKG = "extra_pkg", EXTRA_VERSION_NAME = "extra_version_name", EXTRA_VERSION_CODE = "extra_version_code";

    private ActivityAppDetailsBinding binding;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        binding = ActivityAppDetailsBinding.inflate(getLayoutInflater());
        setContentView(binding.getRoot());
        
        String title = getIntent().getStringExtra(EXTRA_TITLE);
        binding.tvAppTitle.setText(title);
        binding.tvPackageName.setText(getIntent().getStringExtra(EXTRA_PKG));
        Glide.with(this).load(getIntent().getStringExtra(EXTRA_ICON)).into(binding.ivAppIcon);
        
        String desc = getIntent().getStringExtra(EXTRA_DESC);
        if (desc != null && desc.startsWith("http")) {
            binding.btnReadMore.setVisibility(View.VISIBLE);
            binding.btnReadMore.setOnClickListener(v -> startActivity(new Intent(Intent.ACTION_VIEW, Uri.parse(desc))));
        } else {
            binding.tvAppDescription.setText(desc);
        }
    }

    @Override
    protected void onResume() {
        super.onResume();
        updateActionButtons();
    }

    private void updateActionButtons() {
        String pkg = getIntent().getStringExtra(EXTRA_PKG);
        String url = getIntent().getStringExtra(EXTRA_URL);
        String versionName = getIntent().getStringExtra(EXTRA_VERSION_NAME);
        int latestCode = getIntent().getIntExtra(EXTRA_VERSION_CODE, 0);

        int installedCode = UpdateChecker.getInstalledVersionCode(this, pkg);
        boolean isInstalled = installedCode != -1;
        boolean isUpdate = isInstalled && latestCode > installedCode;

        if (isUpdate) binding.btnDownload.setText("Update Available (v" + versionName + ")");
        else if (isInstalled) binding.btnDownload.setText("Open App");
        else binding.btnDownload.setText("Download & Install");

        binding.btnDownload.setOnClickListener(v -> {
            if (isUpdate || !isInstalled) {
                Intent intent = new Intent(this, YOUR_PACKAGE_NAME.ui.update.UpdateDownloadActivity.class);
                intent.putExtra("apk_url", url);
                intent.putExtra("version_name", versionName);
                intent.putExtra("app_title", getIntent().getStringExtra(EXTRA_TITLE));
                startActivity(intent);
            } else {
                startActivity(getPackageManager().getLaunchIntentForPackage(pkg));
            }
        });
    }
}
```

---

## 6. XML Layouts

### `activity_update_download.xml`
```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    android:padding="24dp">

    <TextView android:id="@+id/tvUpdateTitle" android:layout_width="wrap_content" android:layout_height="wrap_content" android:text="Downloading Update" android:textSize="18sp" android:textStyle="bold" />
    <TextView android:id="@+id/tvUpdateVersion" android:layout_width="wrap_content" android:layout_height="wrap_content" android:layout_marginTop="4dp" android:textSize="12sp" />
    <com.google.android.material.progressindicator.LinearProgressIndicator android:id="@+id/progressDownload" android:layout_width="match_parent" android:layout_height="wrap_content" android:layout_marginTop="24dp" app:trackThickness="8dp" />
    <TextView android:id="@+id/tvProgressDetails" android:layout_width="wrap_content" android:layout_height="wrap_content" android:layout_gravity="end" android:layout_marginTop="8dp" android:text="0% (0 MB / 0 MB)" />
    <com.google.android.material.button.MaterialButton android:id="@+id/btnInstallUpdate" android:layout_width="match_parent" android:layout_height="wrap_content" android:layout_marginTop="24dp" android:text="Install Update" android:enabled="false" />
</LinearLayout>
```

### `item_app_store.xml`
```xml
<com.google.android.material.card.MaterialCardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent" android:layout_height="wrap_content" android:layout_margin="8dp" app:cardCornerRadius="16dp">
    <LinearLayout android:layout_width="match_parent" android:layout_height="wrap_content" android:orientation="horizontal" android:padding="16dp">
        <ImageView android:id="@+id/iv_app_icon" android:layout_width="64dp" android:layout_height="64dp" />
        <LinearLayout android:layout_width="0dp" android:layout_height="wrap_content" android:layout_weight="1" android:orientation="vertical" android:layout_marginStart="16dp">
            <TextView android:id="@+id/tv_app_title" android:layout_width="wrap_content" android:layout_height="wrap_content" android:textStyle="bold" />
            <TextView android:id="@+id/tv_version" android:layout_width="wrap_content" android:layout_height="wrap_content" android:textColor="?attr/colorPrimary" />
            <TextView android:id="@+id/tv_app_description" android:layout_width="wrap_content" android:layout_height="wrap_content" android:maxLines="2" android:ellipsize="end" />
            <TextView android:id="@+id/tv_status" android:layout_width="wrap_content" android:layout_height="wrap_content" android:visibility="gone" android:textStyle="bold" />
        </LinearLayout>
    </LinearLayout>
</com.google.android.material.card.MaterialCardView>
```
