## Packaging a chart

Helm supports packaging a chart directory into a `.tgz` archive. This is useful when you want to distribute a chart, upload it to a chart repository, or install from a packaged chart instead of the source directory.

```bash
helm package ./mongodb
```

That creates a file like `mongodb-0.1.0.tgz` in the current directory. You can then install it with:

```bash
helm install mongodb ./mongodb-0.1.0.tgz -n grade-submission --create-namespace
```

Note:
- `helm install ./mongodb` installs directly from the chart directory and does not create a `.tgz` file.
- A `.tgz` file only appears if you explicitly run `helm package`.