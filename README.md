# webp_animetion_player.php
PHP プログラムで、PNG 連番画像にコード命名規則を持たせるコード
<?php
/**
 * WEBP アニメーション一括変換プロセッサー用 GUI ＆ 処理スクリプト
 */
set_time_limit(600);

$message = "";
$outputLog = "";

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $selectedCategory = $_POST['category'] ?? 'Category 01';
    $selectedFps = (int)($_POST['fps'] ?? 8);
    $targetWidth = 854;
    $targetHeight = 480;
    $quality = 80;

    $baseDir = "D:/xampp/htdocs/Player/MyAnimation";
    $targetDir = $baseDir . DIRECTORY_SEPARATOR . $selectedCategory;

    $fpsMap = [
        8  => ["id" => "7", "max_frames" => 480],
        12 => ["id" => "B", "max_frames" => 720],
        16 => ["id" => "F", "max_frames" => 960],
        24 => ["id" => "0", "max_frames" => 1440]
    ];

    if (!isset($fpsMap[$selectedFps])) {
        $message = "エラー: 未対応のfpsです。";
    } else {
        $fpsId = $fpsMap[$selectedFps]["id"];
        $maxFramesPerMin = $fpsMap[$selectedFps]["max_frames"];
        $validExts = ['.png', '.jpg', '.jpeg', '.webp', '.bmp'];

        if (!is_dir($targetDir)) {
            $message = "エラー: 指定されたフォルダが存在しません: {$targetDir}";
        } else {
            // 再帰的にディレクトリを走査
            $iterator = new RecursiveIteratorIterator(
                new RecursiveDirectoryIterator($targetDir, RecursiveDirectoryIterator::SKIP_DOTS),
                RecursiveIteratorIterator::SELF_FIRST
            );

            $directories = [$targetDir];
            foreach ($iterator as $fileInfo) {
                if ($fileInfo->isDir()) {
                    $directories[] = $fileInfo->getPathname();
                }
            }

            $processedCount = 0;
            ob_start();

            foreach ($directories as $currentDir) {
                $currentPath = rtrim($currentDir, DIRECTORY_SEPARATOR);
                if (str_contains($currentPath, 'output_chunks')) {
                    continue;
                }

                $imgFiles = [];
                $handle = opendir($currentPath);
                if ($handle) {
                    while (($entry = readdir($handle)) !== false) {
                        if ($entry === '.' || $entry === '..') continue;
                        $filePath = $currentPath . DIRECTORY_SEPARATOR . $entry;
                        if (is_file($filePath)) {
                            $ext = strtolower(pathinfo($filePath, PATHINFO_EXTENSION));
                            if (in_array('.' . $ext, $validExts, true)) {
                                $imgFiles[] = $filePath;
                            }
                        }
                    }
                    closedir($handle);
                }

                if (empty($imgFiles)) continue;
                sort($imgFiles);

                $outDir = $currentPath . DIRECTORY_SEPARATOR . "output_chunks";
                if (!is_dir($outDir)) {
                    mkdir($outDir, 0777, true);
                }

                $relativePath = str_replace($targetDir, '', $currentPath);
                echo "[" . $selectedCategory . ($relativePath === '' ? '' : " -> " . trim($relativePath, DIRECTORY_SEPARATOR)) . "] 変換開始: " . count($imgFiles) . "枚<br>\n";

                foreach ($imgFiles as $index => $filePath) {
                    $currentMinute = intdiv($index, $maxFramesPerMin);
                    $frameInMinute = $index % $maxFramesPerMin;

                    if ($currentMinute > 299) {
                        echo "  &nbsp;&nbsp;警報: 300分制限を超えたため " . htmlspecialchars(basename($filePath)) . " 以降をスキップ<br>\n";
                        break;
                    }

                    $newFilename = sprintf("%s%03d%04d.webp", $fpsId, $currentMinute, $frameInMinute);
                    $destFile = $outDir . DIRECTORY_SEPARATOR . $newFilename;

                    if (file_exists($destFile)) {
                        continue;
                    }

                    try {
                        $ext = strtolower(pathinfo($filePath, PATHINFO_EXTENSION));
                        $sourceImg = match($ext) {
                            'png' => imagecreatefrompng($filePath),
                            'jpg', 'jpeg' => imagecreatefromjpeg($filePath),
                            'webp' => imagecreatefromwebp($filePath),
                            'bmp' => imagecreatefrombmp($filePath),
                            default => null
                        };

                        if (!$sourceImg) continue;

                        $resizedImg = imagescale($sourceImg, $targetWidth, $targetHeight, IMG_BILINEAR_FIXED);
                        if ($resizedImg) {
                            imagewebp($resizedImg, $destFile, $quality);
                            imagedestroy($resizedImg);
                            $processedCount++;
                        }
                        imagedestroy($sourceImg);
                    } catch (Exception $e) {
                        echo "  &nbsp;&nbsp;エラー: " . htmlspecialchars(basename($filePath)) . "<br>\n";
                    }
                }
            }
            $outputLog = ob_get_clean();
            $message = "変換処理が完了しました！（新規変換数: {$processedCount} 枚）";
        }
    }
}
?>
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>WebP Animation Processor GUI</title>
    <style>
        body { font-family: sans-serif; background: #f4f6f9; color: #333; margin: 0; padding: 20px; }
        .container { max-width: 800px; margin: 0 auto; background: #fff; padding: 30px; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.05); }
        h1 { font-size: 20px; border-bottom: 2px solid #007bff; padding-bottom: 10px; margin-top: 0; }
        .form-group { margin-bottom: 20px; }
        label { display: block; font-weight: bold; margin-bottom: 5px; }
        select, button { width: 100%; padding: 10px; font-size: 16px; border: 1px solid #ccc; border-radius: 4px; box-sizing: border-box; }
        button { background: #007bff; color: #fff; border: none; cursor: pointer; font-weight: bold; margin-top: 10px; }
        button:hover { background: #0056b3; }
        .message { padding: 15px; margin-bottom: 20px; border-radius: 4px; background: #e2f0d9; color: #385723; }
        .log-box { background: #1e1e1e; color: #d4d4d4; padding: 15px; border-radius: 4px; font-family: monospace; font-size: 13px; max-height: 300px; overflow-y: auto; white-space: pre-wrap; }
    </style>
</head>
<body>
<div class="container">
    <h1>WebP Animation Converter GUI</h1>
    
    <?php if (!empty($message)): ?>
        <div class="message"><?php echo htmlspecialchars($message); ?></div>
    <?php endif; ?>

    <form method="POST">
        <div class="form-group">
            <label for="category">対象カテゴリを選択</label>
            <select name="category" id="category">
                <?php for ($i = 1; $i <= 7; $i++): ?>
                    <?php $catName = sprintf("Category %02d", $i); ?>
                    <option value="<?php echo $catName; ?>" <?php echo (isset($_POST['category']) && $_POST['category'] === $catName) ? 'selected' : ''; ?>>
                        <?php echo $catName; ?>
                    </option>
                <?php endfor; ?>
            </select>
        </div>

        <div class="form-group">
            <label for="fps">フレームレート (FPS) を選択</label>
            <select name="fps" id="fps">
                <option value="8" <?php echo (isset($_POST['fps']) && $_POST['fps'] == 8) ? 'selected' : 'selected'; ?>>8 fps (識別子: 7 / チャンク上限: 480フレーム)</option>
                <option value="12" <?php echo (isset($_POST['fps']) && $_POST['fps'] == 12) ? 'selected' : ''; ?>>12 fps (識別子: B / チャンク上限: 720フレーム)</option>
                <option value="16" <?php echo (isset($_POST['fps']) && $_POST['fps'] == 16) ? 'selected' : ''; ?>>16 fps (識別子: F / チャンク上限: 960フレーム)</option>
                <option value="24" <?php echo (isset($_POST['fps']) && $_POST['fps'] == 24) ? 'selected' : ''; ?>>24 fps (識別子: 0 / チャンク上限: 1440フレーム)</option>
            </select>
        </div>

        <button type="submit">変換処理を実行する</button>
    </form>

    <?php if (!empty($outputLog)): ?>
        <h3 style="margin-top: 30px;">実行ログ</h3>
        <div class="log-box"><?php echo $outputLog; ?></div>
    <?php endif; ?>
</div>
</body>
</html>