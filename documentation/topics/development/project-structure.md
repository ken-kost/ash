# Project Structure

In this guide we'll discuss some best practices for how to structure your project. These recommendations align well with [Elixir conventions](https://hexdocs.pm/elixir/1.16.2/naming-conventions.html#casing) around file and module naming. These conventions allow for a logical coupling of module and file names, and help keep your project organized and easy to navigate.

> ### These are recommendations {: .info}
>
> None of the things we show you here are _requirements_, only recommendations.
> Feel free to plot your own course here. Ash avoids any pattern that requires
> you to name a  file or module in a specific way, or put them in a specific
> place. This ensures that all connections between one module and another
> module are _explicit_ rather than _implicit_.

These recommendations all correspond to standard practice in most Elixir/Phoenix applications

```
lib/
├── my_app/                    # Your application's main namespace
│   ├── accounts.ex            # Accounts domain module
│   ├── helpdesk.ex            # Helpdesk domain module
│   │
│   ├── accounts/               # Accounts context
│   │   ├── user.ex             # User resource
│   │   ├── user/               # User resource files
│   │   ├── token.ex            # Token resource
│   │   └── password_helper.ex  # Support module
│   │
│   └── helpdesk/            # Helpdesk context
│       ├── ticket.ex        # Ticket resource
│       ├── notification.ex  # Notification resource
│       ├── other_file.ex    # Support module
│       └── ticket/          # Ticket resource files
│           ├── preparations/
│           ├── changes/
│           └── checks/
```

Place your Ash application in the standard Elixir application directory `lib/my_app`. Your `Ash.Domain` modules should be at the root level of this directory. Each domain should have a directory named after it, containing the domain's `Ash.Resource` modules and any of the domain's supporting modules. All resource interaction ultimately goes through a domain module.

For resources that require additional files, create a dedicated folder in the domain context named after the resource. We suggest organizing these supplementary files into subdirectories by type (like `changes/`, `preparations/`, etc.), though this organization is optional.

# Where do I put X thing

The purpose of Ash is to be both the model of and the interface to your domain logic (A.K.A business logic). Applying this generally looks like building as much of your domain logic "behind" your resources. This does not mean, however, that everything has to go _inside of_ your resources. For example, if you have a `Purchase` resource, and you want to be able to display a list of purchases that were taxable, and also calculate the percentage of the purchase that was taxable. You might have an action called `:taxable` and a calculation called `:percentage_tax`.

## Example 1: Reads & Calculations

```elixir
actions do
  ...

  read :taxable do
    filter expr(taxable == true)
  end
end

calculations do
  calculate :percentage_tax, :decimal, expr(
    sum(line_items, field: :amount, query: [filter: tax == true]) /
    sum(line_items, field: :amount)
  )
end
```

In practice, you may not need the `taxable` action, i.e perhaps you simply want a "taxable" checkbox on a list view in your application, in which case you may use the primary read, or some other read like `:transaction_report`. You would then, on the consumer, provide the filter for `taxable == true`, and load the `:percentage_tax` calculation.

## Example 2: Using external data in create actions

Lets say you want the user to fill in a github issue id, and you will fetch information from that github issue to use as part of creating a "ticket" in your system.. You might be tempted to do something like this in a LiveView:

```elixir
def handle_event("link_ticket", %{"issue_id" => issue_id}, socket) do
  issue_info = GithubApi.get_issue(issue_id)

  MyApp.Support.update_ticket(socket.assigns.ticket_id, %{issue_info: %{
    title: issue_info.title,
    body: issue_info.body
  }})
end
```

But this is putting business logic inside of your UI/representation layer. Instead, you should write an action and put this logic inside of it.

```elixir
defmodule MyApp.Ticket.FetchIssueInfo do
  use Ash.Resource.Change

  def change(changeset, _, _) do
    Ash.Changeset.before_transaction(changeset, fn changeset ->
      issue_info = GithubApi.get_issue(changeset.arguments.issue_id)

      Ash.Changeset.force_change_attributes(changeset, %{
        issue_info: %{
          title: issue_info.title,
          body: issue_info.body
        }
      })
    end)
  end
end
```

Then you'd have an action like this:

```elixir
update :link_ticket do
  argument :issue_id, :string, allow_nil?: false

  change MyApp.Ticket.FetchIssueInfo
end
```

This cleanly encapsulates the operation behind the resource, even while the code for fetching the github issue still lives in a `GitHubApi` module.
