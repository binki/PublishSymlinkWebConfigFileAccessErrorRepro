This project demonstrates a failure in web publish that occurs when
using hardlinks for publish.

To reduce storage usage, there are [multiple properties which control
usage of hard links](https://github.com/dotnet/msbuild/issues/3788)
(for which I cannot find proper documentation!). These are:

* `CreateHardLinksForCopyFilesToOutputDirectoryIfPossible`
* `CreateHardLinksForCopyAdditionalFilesIfPossible`
* `CreateHardLinksForCopyLocalIfPossible`
* `CreateHardLinksForPublishFilesIfPossible`

One can set these all `true` in `Directory.Build.props`. The benefit
is that the multiple copies of binaries in `bin/` folders in a large
solution in which many projects depend on each other can all refer to
the same file, taking up less disk space.

Lots of unix tools use this technique to speed up copies (by creating
additional links instad of actually copying) and this behavior is
expected. However, in the Windows world, this is still not normal. And
tooling does not always support it well because it is still off by
default, further perpetuating the idea that this is abnormal on
Windows.

The `Microsoft.NET.Sdk.Web` MSBuild SDK falls into this latter
category of tooling which does not support this well. If you have a
`web.config` in your project and attempt to publish with
`CreateHardLinksForPublishFilesIfPossible` set to `true`, the publish
process will perform these steps during the publish:

1. Create a hardlink named `«publishDir»\web.config` referring to `«projectDir»\web.config`.
2. Execute `web.config` a transformations task which attempts to open `«projectDir»\web.config` for reading and, prior to closing it, attempts to open `«publishDir»\web.config` for writing.

To reproduce this:

1. Clone this project.
2. Build it once.
3. Use the Publish to Local Folder publishing option.
4. Encounter an error:

   ```
   C:\Program Files\dotnet\sdk\6.0.302\Sdks\Microsoft.NET.Sdk.Publish\targets\TransformTargets\Microsoft.NET.Sdk.Publish.TransformFiles.targets(50,5): Error MSB4018: The "TransformWebConfig" task failed unexpectedly.
System.IO.IOException: The process cannot access the file 'C:\Users\ohnob\AppData\Local\Temp\PublishSymlinkWebConfigFileAccessErrorRepro\PublishSymlinkWebConfigFileAccessErrorRepro\obj\Release\net6.0\PubTmp\Out\web.config' because it is being used by another process.
      at System.IO.__Error.WinIOError(Int32 errorCode, String maybeFullPath)
      at System.IO.File.InternalCopy(String sourceFileName, String destFileName, Boolean overwrite, Boolean checkHost)
      at Microsoft.NET.Sdk.Publish.Tasks.TransformWebConfig.Execute()
      at Microsoft.Build.BackEnd.TaskExecutionHost.Microsoft.Build.BackEnd.ITaskExecutionHost.Execute()
      at Microsoft.Build.BackEnd.TaskBuilder.<ExecuteInstantiatedTask>d__26.MoveNext()
   ```

Fortunately, this fails and nothing is hurt. However, if the
transformation task closed `«projectDir»\web.config` prior to opening
«publishDir»\web.config` for writing, we would have more trouble—the
transformations, which should not be included in source control, would
be written to `«publishDir»\web.config` and, likely, accidentally get
committed to source control!

The correct way to handle this situation is to, like unix tools,
ensure that hardlinks are broken. Do this by:

1. Open a new, temporary file.
2. Open and read the `«projectDir»\web.config`.
3. Execute the transformation.
4. Close `«projectDir»\web.config`.
5. Close the new, temporary file.
6. Rename the new, temporary file to `«publishDir»\web.config`.

This last step breaks the link.
