# jinjaform

**Syntax highlighting for Jinja templated Terraform files**

This is a very basic syntax highlighter (and formatting hack) for jinja wrappers to terraform code - you can use jinja to conditionally render terraform file contents, or dynamically render provisioners and providers within terraform.

## Requirements

- Expects syntax highlighting `source.hcl.terraform` from [Hashicorp Terraform](https://marketplace.visualstudio.com/items?itemName=HashiCorp.terraform) extension.  
- Expects syntax highlighting `source.jinja` from [Jinja](https://marketplace.visualstudio.com/items?itemName=wholroyd.jinja) extension.  

When installing through the VSCode Marketplace these should be installed automatically. Kindly verify these are enabled before raising issues.

## Features

This is useful for syntax highlighting in projects where you want to abstract terraform with jinja and use a yaml to render terraform code based on user input.  

For example; with [jinja cli](https://pypi.org/project/jinja-cli/) installed, and files as in [`examples/`](./examples/) folder, you could generate/update the main.tf dynamically as

```bash
jinja -d config.yml -f yaml main.tf.j2 --out generated_main.tf
```

It also adds a language capability called `jinjaform` that can be used to configure custom formatting logic

## Formatting

By default, it associates all files with extensions `.tf.j2`, `.tf.jinja`, `.tf.jinja2`, `.tfvars.j2`, `.tfvars.jinja`, `.tfvars.jinja2`, `.hcl.j2`, `.hcl.jinja`, `.hcl.jinja2` to the new language `jinjaform`.
Assuming you want to format the file based on `terraform` rules, and assuming you are only using `{% if ... %}`, `{% for ... %}` or `{% with ... %}` and `{% include ... %}` features from jinja, you could use [Faster Format with CLI](https://marketplace.visualstudio.com/items?itemName=djpai.faster-format-with-cli) extension and set up the following language rule in your VSCode [settings.json](https://code.visualstudio.com/docs/getstarted/settings#_settingsjson)

```json
{
  // ...
  "[jinjaform]": {
    "editor.defaultFormatter": "djpai.faster-format-with-cli",
    "djpai.format.command": "sed '/^$/N;/^\\n$/D' | sed -e 's/{%/#{%/g' | terraform fmt -write=false - | sed -e 's/#{%/{%/g'",
    "djpai.format.mode": "inline_stdin"
  }
  // ...
}
```

We are using the `djpai.format.command` setting provided by the extension to send the contents of our file to the terraform formatter.  
In the command, we are using [sed](https://www.gnu.org/software/sed/manual/sed.html#Overview) to remove any extraneous newlines from the file; comment and uncomment the jinja specific keywords that could cause syntax issues in terraform, and finally formatting with terraform fmt and reverting back changes.  

You may have to tweak the regexes based on how you are leveraging jinja features when templating terraform.

> [!NOTE]
>
> - You can debug your format command through the __OUTPUT__ Panel
> `View > Output` or `Cmd/Ctrl + Shift + P > View: Toggle Output` in VSCode and selecting the `fasterFormatWithCLI` channel.
> - You may also need to `Developer: Set Log Level...` as `Debug` to get more logs

## Known Issues

1. Nested jinja blocks will not respect highlighting
2. No language server capabilities of terraform is possible since it requires custom clients and request forwarding, which is beyond the scope for now.
Refer [VSCode/Embedded Languages](https://code.visualstudio.com/api/language-extensions/embedded-languages#request-forwarding), [microsoft code samples](https://github.com/microsoft/vscode-extension-samples/blob/5651637c527e07173bd066f5fb7e171ef4616cab/lsp-embedded-request-forwarding/client/src/extension.ts#L50) and this [SO answer](https://stackoverflow.com/a/48995978/8453502) to understand the complexity.  
3. Commenting is opinionated, since the syntax for commenting in jinja is different from terraform. By design, the comments for jinjaform uses terraform commenting `# inline` and `/* block */` since the expectation is that it will be rendered into terraform for use.

## Credits

- Got idea from the [Better Jinja](https://marketplace.visualstudio.com/items?itemName=samuelcolvin.jinjahtml) extension and its [jinja-terraform](https://github.com/samuelcolvin/jinjahtml-vscode/blob/main/syntaxes/jinja-terraform.tmLanguage.json) support. But it had bugs with the jinja [folding logic](https://github.com/samuelcolvin/jinjahtml-vscode/issues/153) and the language pattern was changed by Hashicorp from `source.terraform` to `source.hcl.terraform` that broke this extension

- [Reference for Language grammar config](https://macromates.com/manual/en/language_grammars)