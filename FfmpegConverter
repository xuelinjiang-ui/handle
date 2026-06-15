using System.Diagnostics;
using System.IO;

namespace WpfApp1.Services;

public interface IFfmpegConverter
{
    bool IsAvailable { get; }

    string FfmpegDirectory { get; }

    Task<ConvertResult> ConvertToMp3Async(string inputPath, string outputPath);
}

public sealed class FfmpegConverter : IFfmpegConverter
{
    private const string ToolDirectoryName = "Tools";
    private const string ExecutableName = "ffmpeg.exe";
    private const string AudioCodec = "libmp3lame";
    private const string AudioBitrate = "192k";

    private readonly string _ffmpegPath;

    public FfmpegConverter()
        : this(AppDomain.CurrentDomain.BaseDirectory)
    {
    }

    internal FfmpegConverter(string applicationBaseDirectory)
    {
        if (string.IsNullOrWhiteSpace(applicationBaseDirectory))
        {
            throw new ArgumentException("Value cannot be null or whitespace.", nameof(applicationBaseDirectory));
        }

        _ffmpegPath = Path.Combine(applicationBaseDirectory, ToolDirectoryName, ExecutableName);
    }

    public bool IsAvailable => File.Exists(_ffmpegPath);

    public string FfmpegDirectory => Path.GetDirectoryName(_ffmpegPath)!;

    public async Task<ConvertResult> ConvertToMp3Async(string inputPath, string outputPath)
    {
        if (!IsAvailable)
        {
            return ConvertResult.Failure("ffmpeg.exe was not found: " + FfmpegDirectory);
        }

        try
        {
            if (string.IsNullOrWhiteSpace(inputPath))
            {
                throw new ArgumentException("Value cannot be null or whitespace.", nameof(inputPath));
            }

            if (string.IsNullOrWhiteSpace(outputPath))
            {
                throw new ArgumentException("Value cannot be null or whitespace.", nameof(outputPath));
            }

            if (!File.Exists(inputPath))
            {
                return ConvertResult.Failure("Decoded input file does not exist: " + inputPath);
            }

            string? outputDirectory = Path.GetDirectoryName(outputPath);
            if (string.IsNullOrWhiteSpace(outputDirectory))
            {
                return ConvertResult.Failure("Invalid output path: " + outputPath);
            }

            Directory.CreateDirectory(outputDirectory);

            if (NeedsPathStaging(inputPath) || NeedsPathStaging(outputPath))
            {
                return await ConvertWithStagingAsync(inputPath, outputPath);
            }

            return await ConvertDirectAsync(inputPath, outputPath);
        }
        catch (Exception exception)
        {
            return ConvertResult.Failure("Failed to start FFmpeg: " + exception.Message);
        }
    }

    private async Task<ConvertResult> ConvertDirectAsync(string inputPath, string outputPath)
    {
        using Process process = StartProcess(inputPath, outputPath);
        string standardError = await process.StandardError.ReadToEndAsync();
        await ProcessCompat.WaitForExitAsync(process);

        return BuildConvertResult(outputPath, process.ExitCode, standardError);
    }

    private async Task<ConvertResult> ConvertWithStagingAsync(string inputPath, string outputPath)
    {
        string stagingDirectory = Path.Combine(
            Path.GetTempPath(),
            "WpfApp1",
            "ffmpeg",
            Guid.NewGuid().ToString("N"));

        Directory.CreateDirectory(stagingDirectory);

        string stagedInputPath = Path.Combine(stagingDirectory, "input" + Path.GetExtension(inputPath));
        string stagedOutputPath = Path.Combine(stagingDirectory, "output.mp3");

        try
        {
            File.Copy(inputPath, stagedInputPath, overwrite: true);

            ConvertResult stagedResult = await ConvertDirectAsync(stagedInputPath, stagedOutputPath);
            if (!stagedResult.IsSuccess)
            {
                return stagedResult;
            }

            File.Copy(stagedOutputPath, outputPath, overwrite: true);
            return ConvertResult.Success(outputPath);
        }
        finally
        {
            TryDeleteDirectory(stagingDirectory);
        }
    }

    private Process StartProcess(string inputPath, string outputPath)
    {
        ProcessStartInfo startInfo = CreateProcessStartInfo(inputPath, outputPath);
        return Process.Start(startInfo) ?? throw new InvalidOperationException("Unable to start ffmpeg.exe.");
    }

    private ProcessStartInfo CreateProcessStartInfo(string inputPath, string outputPath)
    {
        return new ProcessStartInfo
        {
            FileName = _ffmpegPath,
            Arguments = ProcessCompat.BuildArguments(
                "-y",
                "-i",
                inputPath,
                "-codec:a",
                AudioCodec,
                "-b:a",
                AudioBitrate,
                "-map_metadata",
                "0",
                outputPath),
            UseShellExecute = false,
            CreateNoWindow = true,
            RedirectStandardError = true
        };
    }

    private static ConvertResult BuildConvertResult(string outputPath, int exitCode, string standardError)
    {
        if (exitCode == 0 && IsValidOutputFile(outputPath))
        {
            return ConvertResult.Success(outputPath);
        }

        string errorDetails = string.IsNullOrWhiteSpace(standardError)
            ? string.Empty
            : Environment.NewLine + standardError.Trim();

        return ConvertResult.Failure("FFmpeg did not create an MP3 file. Exit code: " + exitCode + errorDetails);
    }

    private static bool IsValidOutputFile(string outputPath)
    {
        return File.Exists(outputPath) && new FileInfo(outputPath).Length > 0;
    }

    private static bool NeedsPathStaging(string path)
    {
        foreach (char character in path)
        {
            if (character > 0x7F || character == '"')
            {
                return true;
            }
        }

        return false;
    }

    private static void TryDeleteDirectory(string path)
    {
        if (string.IsNullOrWhiteSpace(path) || !Directory.Exists(path))
        {
            return;
        }

        try
        {
            Directory.Delete(path, recursive: true);
        }
        catch
        {
        }
    }
}
