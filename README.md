package com.umbanda.feitura;

import android.Manifest;
import android.content.Context;
import android.content.Intent;
import android.content.pm.PackageManager;
import android.net.Uri;
import android.os.Build;
import android.os.Bundle;
import android.os.Environment;
import android.view.KeyEvent;
import android.view.View;
import android.webkit.JavascriptInterface;
import android.webkit.ValueCallback;
import android.webkit.WebChromeClient;
import android.webkit.WebSettings;
import android.webkit.WebView;
import android.webkit.WebViewClient;
import android.widget.ProgressBar;
import android.widget.Toast;

import androidx.annotation.NonNull;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.app.ActivityCompat;
import androidx.core.content.ContextCompat;
import androidx.core.content.FileProvider;

import org.json.JSONObject;

import java.io.*;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Locale;

public class MainActivity extends AppCompatActivity {
    private WebView webView;
    private ProgressBar progressBar;
    private static final int PERMISSION_REQUEST_CODE = 100;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        
        requestPermissions();
        
        webView = findViewById(R.id.webview);
        progressBar = findViewById(R.id.progressBar);
        
        configureWebView();
        
        // Verifica se foi aberto para editar arquivo HTML
        Intent intent = getIntent();
        if (Intent.ACTION_VIEW.equals(intent.getAction())) {
            Uri uri = intent.getData();
            if (uri != null) {
                webView.loadUrl(uri.toString());
                return;
            }
        }
        
        // Carrega o HTML padrão dos assets
        webView.loadUrl("file:///android_asset/minha-feitura-umbanda.html");
    }

    private void configureWebView() {
        WebSettings webSettings = webView.getSettings();
        
        webSettings.setJavaScriptEnabled(true);
        webSettings.setDomStorageEnabled(true);
        webSettings.setDatabaseEnabled(true);
        webSettings.setAllowFileAccess(true);
        webSettings.setAllowContentAccess(true);
        webSettings.setAllowFileAccessFromFileURLs(true);
        webSettings.setAllowUniversalAccessFromFileURLs(true);
        
        String databasePath = this.getApplicationContext()
            .getDir("database", MODE_PRIVATE).getPath();
        webSettings.setDatabasePath(databasePath);
        
        webSettings.setBuiltInZoomControls(false);
        webSettings.setDisplayZoomControls(false);
        webSettings.setLoadWithOverviewMode(true);
        webSettings.setUseWideViewPort(true);
        
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            webSettings.setMixedContentMode(WebSettings.MIXED_CONTENT_ALWAYS_ALLOW);
        }
        
        // Interface JavaScript completa
        webView.addJavascriptInterface(new WebAppInterface(this), "Android");
        
        webView.setWebViewClient(new WebViewClient() {
            @Override
            public void onPageStarted(WebView view, String url, android.graphics.Bitmap favicon) {
                progressBar.setVisibility(View.VISIBLE);
            }
            
            @Override
            public void onPageFinished(WebView view, String url) {
                progressBar.setVisibility(View.GONE);
                
                // Injeta script para capturar dados do localStorage
                view.evaluateJavascript(
                    "javascript:(function() {" +
                    "  var data = {" +
                    "    items: localStorage.getItem('feitura_items') || '{}'," +
                    "    status: localStorage.getItem('feitura_status') || '{}'," +
                    "    listas: localStorage.getItem('feitura_listas') || '[]'," +
                    "    editado: localStorage.getItem('feitura_editado') || 'false'" +
                    "  };" +
                    "  Android.onDataLoaded(JSON.stringify(data));" +
                    "})()",
                    null
                );
            }
        });
        
        webView.setWebChromeClient(new WebChromeClient() {
            @Override
            public void onProgressChanged(WebView view, int newProgress) {
                progressBar.setProgress(newProgress);
            }
        });
    }
    
    public class WebAppInterface {
        Context mContext;
        
        WebAppInterface(Context c) {
            mContext = c;
        }
        
        @JavascriptInterface
        public void showToast(String message) {
            Toast.makeText(mContext, message, Toast.LENGTH_SHORT).show();
        }
        
        @JavascriptInterface
        public void onDataLoaded(String jsonData) {
            // Dados carregados do localStorage
        }
        
        @JavascriptInterface
        public void exportHTML(String htmlContent) {
            saveEditedHTML(htmlContent);
        }
        
        @JavascriptInterface
        public void exportText(String textContent) {
            saveTextFile(textContent);
        }
        
        @JavascriptInterface
        public void shareText(String text) {
            shareContent(text, "text/plain");
        }
        
        @JavascriptInterface
        public void shareHTML(String html) {
            shareContent(html, "text/html");
        }
        
        @JavascriptInterface
        public String getEditTimestamp() {
            SimpleDateFormat sdf = new SimpleDateFormat("dd/MM/yyyy HH:mm", Locale.getDefault());
            return sdf.format(new Date());
        }
    }
    
    private void saveEditedHTML(String htmlContent) {
        try {
            String timestamp = new SimpleDateFormat("yyyyMMdd_HHmmss", Locale.getDefault())
                .format(new Date());
            String filename = "Minha_Feitura_EDITADO_" + timestamp + ".html";
            
            File downloadsDir = Environment.getExternalStoragePublicDirectory(
                Environment.DIRECTORY_DOWNLOADS);
            File file = new File(downloadsDir, filename);
            
            // Adiciona os dados do localStorage no HTML
            String finalHTML = injectLocalStorageIntoHTML(htmlContent);
            
            FileOutputStream fos = new FileOutputStream(file);
            fos.write(finalHTML.getBytes("UTF-8"));
            fos.close();
            
            // Compartilha automaticamente
            shareFile(file);
            
            Toast.makeText(this, 
                "HTML editado salvo com sucesso!", Toast.LENGTH_LONG).show();
                
        } catch (IOException e) {
            Toast.makeText(this, 
                "Erro ao salvar: " + e.getMessage(), Toast.LENGTH_LONG).show();
        }
    }
    
    private String injectLocalStorageIntoHTML(String htmlContent) {
        // Injeta os dados do localStorage como script no HTML
        String injectionScript = 
            "<script>\n" +
            "// Dados salvos da edição\n" +
            "document.addEventListener('DOMContentLoaded', function() {\n" +
            "  try {\n" +
            "    var savedData = JSON.parse(localStorage.getItem('feitura_backup'));\n" +
            "    if (savedData) {\n" +
            "      localStorage.setItem('feitura_items', savedData.items);\n" +
            "      localStorage.setItem('feitura_status', savedData.status);\n" +
            "      localStorage.setItem('feitura_listas', savedData.listas);\n" +
            "      location.reload();\n" +
            "    }\n" +
            "  } catch(e) {}\n" +
            "});\n" +
            "</script>\n</body>";
        
        return htmlContent.replace("</body>", injectionScript);
    }
    
    private void saveTextFile(String textContent) {
        try {
            String timestamp = new SimpleDateFormat("yyyyMMdd_HHmmss", Locale.getDefault())
                .format(new Date());
            String filename = "Minha_Feitura_" + timestamp + ".txt";
            
            File downloadsDir = Environment.getExternalStoragePublicDirectory(
                Environment.DIRECTORY_DOWNLOADS);
            File file = new File(downloadsDir, filename);
            
            FileOutputStream fos = new FileOutputStream(file);
            fos.write(textContent.getBytes("UTF-8"));
            fos.close();
            
            shareFile(file);
            
            Toast.makeText(this, 
                "Extrato de texto salvo!", Toast.LENGTH_LONG).show();
                
        } catch (IOException e) {
            Toast.makeText(this, 
                "Erro ao salvar: " + e.getMessage(), Toast.LENGTH_LONG).show();
        }
    }
    
    private void shareFile(File file) {
        Uri fileUri = FileProvider.getUriForFile(
            this,
            getPackageName() + ".provider",
            file
        );
        
        Intent shareIntent = new Intent(Intent.ACTION_SEND);
        shareIntent.setType("text/html");
        shareIntent.putExtra(Intent.EXTRA_STREAM, fileUri);
        shareIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
        shareIntent.putExtra(Intent.EXTRA_TEXT, 
            "🙏 Minha Feitura de Umbanda - Lista Editada\n" +
            "🕯️ Abra o arquivo para ver as oferendas!");
        
        startActivity(Intent.createChooser(shareIntent, "Compartilhar via"));
    }
    
    private void shareContent(String content, String type) {
        Intent shareIntent = new Intent(Intent.ACTION_SEND);
        shareIntent.setType(type);
        shareIntent.putExtra(Intent.EXTRA_TEXT, content);
        startActivity(Intent.createChooser(shareIntent, "Compartilhar via"));
    }
    
    private void requestPermissions() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            String[] permissions = {
                Manifest.permission.WRITE_EXTERNAL_STORAGE,
                Manifest.permission.READ_EXTERNAL_STORAGE
            };
            
            for (String permission : permissions) {
                if (ContextCompat.checkSelfPermission(this, permission) 
                    != PackageManager.PERMISSION_GRANTED) {
                    ActivityCompat.requestPermissions(this, permissions, 
                        PERMISSION_REQUEST_CODE);
                    break;
                }
            }
        }
    }
    
    @Override
    public boolean onKeyDown(int keyCode, KeyEvent event) {
        if (keyCode == KeyEvent.KEYCODE_BACK && webView.canGoBack()) {
            webView.goBack();
            return true;
        }
        return super.onKeyDown(keyCode, event);
    }
    
    @Override
    protected void onDestroy() {
        if (webView != null) {
            webView.destroy();
        }
        super.onDestroy();
    }
}
