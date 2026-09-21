# Advanced @63klabs/cache-data Implementation

> This tutorial is still under development. However, the basic structure is listed below. Please be advised that the content is short, may be missing, and may have inaccuracies or typos. If you would like to contribute updates, please submit an [issue via this repository on GitHub](https://github.com/63Klabs/atlantis-tutorials/issues). Be sure to include the page and what content should be added/updated. If you'd be willing to write a few sentences (or more, but be clear and succinct), please do. Thank you for your understanding.

You will also need to ensure a CloudFormation stack named `<prefix>-cache-data-storage` exists as it is required for Application Starter #02. One easy way to check is to run the command from the CLI (replace 'YOUR_PROFILE' and 'acme'):

```
aws cloudformation list-exports --profile YOUR_PROFILE --query "Exports[?starts_with(Name, 'acme-CacheData')]"
```

- **If it returns `[]`** then the stack does not exist and you will need to tell your instructor, supervisor, or account administrator and move on to the [next tutorial](../03-static-website-deployment/README.md).
- **If it returns a JSON list** of Exports, Names, and Values then you are good to continue with this tutorial.

TODO
