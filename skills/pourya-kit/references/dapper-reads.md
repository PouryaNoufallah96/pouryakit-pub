# Dapper reads

`PouryaKit.Persistence.Dapper`: queries in plain SQL with Dapper on a pooled PostgreSQL connection, and typed IDs as Dapper parameters and results. Namespace `PouryaKit.Persistence.Dapper`. A plain-SQL write that joins the Unit of Work uses `IUnitOfWorkConnection` from `PouryaKit.Persistence` instead.

- [Registration](#registration)
- [Reading](#reading)
- [Reading a Page](#reading-a-page)
- [Typed IDs](#typed-ids)
- [Writing with plain SQL in the Unit of Work](#writing-with-plain-sql-in-the-unit-of-work)
- [Testing](#testing)

The Proving Sample's `GetProduct` and `ListProducts` Features:

```csharp
using Dapper;
using PouryaKit.Cqrs;
using PouryaKit.Domain;
using PouryaKit.Persistence.Dapper;

// Products/Features/GetProduct/
internal sealed record ProductDto(Guid Id, Guid CategoryId, string Name);

// Plain SQL reads no query filter, so it compares the tenant the lease bound itself, as it names is_deleted.
internal sealed class GetProductQueryHandler(IReadConnectionFactory connections)
    : QueryHandler<GetProductQuery, ProductDto>
{
    public override async Task<Result<ProductDto>> Handle(GetProductQuery query, CancellationToken cancellationToken)
    {
        await using var connection = await connections.OpenAsync(cancellationToken);
        var product = await connection.QuerySingleOrDefaultAsync<ProductDto>(
            new CommandDefinition(
                "select id, category_id as categoryid, name from catalog.products where id = @Id and tenant = current_setting('pouryakit.tenant') and is_deleted = false",
                new { Id = new ProductId(query.ProductId) },   // a typed ID as a parameter
                cancellationToken: cancellationToken
            )
        );
        return product is null ? ProductErrors.NotFound : product;
    }
}

// Products/Features/ListProducts/: typed IDs as results, one Page at a time
internal sealed record ProductListItemDto(ProductId Id, CategoryId CategoryId, string Name);

internal sealed class ListProductsQueryHandler(IReadConnectionFactory connections)
    : QueryHandler<ListProductsQuery, Page<ProductListItemDto>>
{
    // The Feature writes its WHERE, the tenant the lease bound included, since plain SQL reads no query filter; each
    // declared filter arrives as @{field}_{operator}, null when not asked for.
    private const string From = """
        from catalog.products p
        where p.tenant = current_setting('pouryakit.tenant')
          and not p.is_deleted
          and (@name_eq is null or p.name = @name_eq)
          and (@name_contains is null or strpos(p.name, @name_contains) > 0)
          and (@category_eq is null or p.category_id = @category_eq)
          and (@created_gte is null or p.created_at >= @created_gte)
        """;

    private static readonly PageSql Sql = new PageSql(
        ListProductsQuery.Paging,
        select: $"select p.id, p.category_id as categoryid, p.name {From}",
        count: $"select count(*) {From}"
    )
        .OrderBy("name", "p.name")
        .OrderBy("created", "p.created_at")
        .ThenBy("p.id");

    public override async Task<Result<Page<ProductListItemDto>>> Handle(ListProductsQuery query, CancellationToken cancellationToken)
    {
        await using var connection = await connections.OpenAsync(cancellationToken);
        return await connection.QueryPageAsync<ProductListItemDto>(Sql, query.Page, cancellationToken: cancellationToken);
    }
}
```

## Registration

- Call `module.AddPouryaKitDapper(connectionName: "app")` in the Composition Root of each Module that reads with Dapper. It reads the connection string `ConnectionStrings:{connectionName}` from the Host's configuration, and makes every typed ID in the assembly of the Module's definition usable with Dapper, or in the assemblies it names instead. An application without Modules names the assembly.
- The Host holds one connection pool, shared by every Module that reads with Dapper, and each scope its own `IReadConnectionFactory`, so resolve it inside a scope, as a handler does. Every Module names the same connection, ignoring case as configuration keys do; naming a different one in the same Host throws while the Host is composed. A connection string the Host's final configuration does not set, or sets blank, fails the Host when it is built, among the composition checks, naming the connection and its key (`Dapper reads use the connection string app, which the Host's configuration does not set (ConnectionStrings:app), …`); nothing connects. In a Host without the composition checks it throws when `IReadConnectionFactory` is first resolved.

## Reading

- Read in a query's handler. Inject `IReadConnectionFactory` and open a connection with `await using var connection = await connections.OpenAsync(cancellationToken);`. Each call leases a new open connection from the pool; the handler owns it and disposes it, which returns it to the pool. Opening with a cancelled token throws `OperationCanceledException`.
- Each lease, as it opens, sets the PostgreSQL session setting `pouryakit.tenant` to the tenant of the scope's Operation with a parameterised `set_config`, read when the lease opens, or to the empty string, which matches no row, where the Operation has no tenant or none began. Npgsql resets a connection's session state when it returns to the pool (`DISCARD ALL`, its default), and every lease sets its own tenant either way, so a lease after another tenant's reads its own tenant's rows. A connection string that sets `Multiplexing=true` is refused, with the composition checks when the Host is built and otherwise at the first lease, since a multiplexed command does not keep the session its lease set. A connection the Kit did not lease has no such setting, and `current_setting('pouryakit.tenant')` fails there.
- A read connection never joins the Operation's Unit of Work. It reads what is committed: inside a command it does not see the Operation's own changes, which the Unit of Work saves only after the handlers return.
- Pass the cancellation token through `new CommandDefinition(sql, parameters, cancellationToken: cancellationToken)`; Dapper's overloads that take the SQL and parameters directly take no token.
- Name each table with its Module's schema (`catalog.products`): the read connection is the Host's, not the Module's. Read only your own Module's schema; another Module's data comes through its Contracts (see [modules.md](modules.md)).
- Columns are in snake_case (see [persistence.md](persistence.md)); alias each to its member name (`category_id as categoryid`). Dapper matches names without regard to case.
- Dapper runs exactly the SQL you write; no EF Core query filter reaches it. A removed soft-deletable aggregate keeps its row, marked `is_deleted = true` (see [persistence.md](persistence.md#audit-and-soft-delete)), so every read of a soft-deletable table states `is_deleted = false` (or `not is_deleted`) itself, as both reads above do. A read that leaves it out returns removed aggregates. In a tenant-scoped Module every read of a tenant-scoped table states `tenant = current_setting('pouryakit.tenant')` the same way, against the tenant the lease bound, never a tenant the handler passes, as both reads above do ([multi-tenancy.md](multi-tenancy.md#reads)). The lease owns that setting: a structure test rejects any SQL literal of a Module that writes session state, one holding `set_config` or a statement that starts with `SET` or `RESET`. State every other predicate the read needs in the SQL too.
- In a tenant-scoped Module, write each read of a tenant-scoped table in the one grammar a structure test accepts: `SELECT list FROM item ((JOIN | INNER JOIN | LEFT JOIN) item ON condition | , item)* WHERE conjunct (AND conjunct)* [ORDER BY list] [LIMIT @p | n] [OFFSET @p | n]`, one statement, each item `[schema.]table [[AS] alias]` with nothing after it, no other `SELECT` or `TABLE` and no set operator, no `OR` or `CASE` outside parentheses in a condition or a conjunct, and for each tenant-scoped item a conjunct that is exactly `p.tenant = current_setting('pouryakit.tenant')` by its alias, or by its name when it has none, or `tenant = current_setting('pouryakit.tenant')` unqualified when the `FROM` has that one item; a parameter such as `@Tenant`, `current_setting` with a second argument or another setting, or the sides swapped is rejected, naming the conjunct it needs. The table of a part a navigation holds in a tenant-scoped aggregate (a part of a part too, and a type derived from a part with a table of its own, whose holder is its base type's table, joined on its primary key) has no tenant column: join each of its items to an item of its holder, itself held so, up to the root held by its conjunct, with `e.ledger_id = l.id`, both sides qualified, for every column of its key to that holder, in the part's own `ON`, or in the holder's when the holder comes later by `JOIN` or `INNER JOIN`, never `LEFT JOIN`, which keeps every row of the part, as for a ledger whose entries are its parts: `select e.amount from accounting.ledger_entries e join accounting.ledgers l on l.id = e.ledger_id where l.tenant = current_setting('pouryakit.tenant')`. A further table that a tenant-scoped entity or a part splits into (`SplitToTable`) is a part of the entity's own table: join it on every primary-key column, `n.id = x.id`, to an item of that table held as above. A part read with no join, joined in the `WHERE` or by a comma, or joined to a root that is not held fails, naming the part's table and its root. Group each `OR` in parentheses, as `ListProducts` does, and hold each tenant-scoped table of a join to the tenant in the `WHERE`, not only in its `ON`. Where a subquery, a set operator or a CTE would read such a table, write a join or a second read. The test rejects every other shape of a string literal that names a tenant-scoped table anywhere in its text and holds a SQL command word, `from` or `join`, a fragment concatenated at run time included, and a block comment, a `$` or a backslash in one; a literal that writes such a table is the SQL-write rule's ([testing.md](testing.md#structure-tests)).
- Read into an `internal` DTO record declared in the Feature, as above, and return a missing row as the Business Area's `NotFound` Error.

## Reading a Page

`ListProducts` above reads one Page of a list. Its query carries a `PageRequest`, declares what a caller may sort and filter by in a `PageDeclaration`, and checks the request in its validator ([commands-queries-validation.md](commands-queries-validation.md#paging)).

- Write the read's SQL whole, once, in a static `PageSql`: the `SELECT` of the items and the `COUNT` of the same selection, sharing one `FROM` and `WHERE`, as `From` above. End the `SELECT` without `order by`, `limit` or a semicolon; the helper appends them.
- Map every sort key the declaration declares to your own column with `OrderBy(sortKey, column)`, and name a unique column with `ThenBy`, usually the id, so rows with equal sort values keep one order from Page to Page. The direction applies to both. A sort key the declaration does not declare is never SQL: validation refuses it, and the helper throws `InvalidOperationException` before any SQL runs if a request reaches it unvalidated.
- Each declared filter field and operator is the parameter `@{field}_{operator}`, the operator by its short name: `@name_eq`, `@name_contains`, `@category_eq`, `@created_gte`. A time arrives in UTC, whatever offset the caller gave. Every one is always bound, to the filter's value as its declared type or to a typed null when the request does not filter on it, so test each in your `WHERE` with `(@x is null or …)`. Use them in both the `SELECT` and the `COUNT`: the `PageSql` constructor throws when either leaves one out, and when a declared filter's value type has no database type, so the total always counts what the items select. A value is only ever a parameter, so SQL in it matches nothing.
- Write each operator's meaning yourself, and keep it the same as an EF Core read of the same declaration: `strpos(p.name, @name_contains) > 0` is a case-sensitive contains with no `LIKE` wildcards, as EF Core's `Contains` is.
- Read with `connection.QueryPageAsync<T>(sql, query.Page, parameters, cancellationToken)` on your lease. Pass the read's own parameters, such as a tenant, in `parameters`; `page_size`, `page_offset` and the filter names are the helper's. It runs the `COUNT`, then the `SELECT`, and answers `Page<T>`; a Page beyond the last has no items and the true total. The two run as separate statements, so a change committed between them can make the total differ from the items by that change.
- Never build SQL from a `PageRequest` yourself: no sort key, field name or value a caller sends belongs in your SQL text.

## Typed IDs

- A typed ID declared in an assembly passed to `AddPouryaKitDapper` works as a parameter and as a result with no code of its own: `new { Id = new ProductId(id) }` sends its value, and a DTO member typed `ProductId` reads its column back into one. Guid, long, int and string IDs all work; a column of another numeric type, such as an integer column read into a long ID, is converted.
- Pass every assembly whose typed IDs you read or write with Dapper; usually that is the Module's own.

## Writing with plain SQL in the Unit of Work

A Module-internal table that no aggregate maps, such as a projection, a read model or a tally, may be written with plain SQL that commits or rolls back with the Operation. Inject `IUnitOfWorkConnection` (`PouryaKit.Persistence`, registered by `UsePouryaKitPostgres`) and pass its transaction to every statement. The Proving Sample's projection is Inventory's copy of Catalog's products, which its Integration Event Handler writes this way from each Snapshot ([messaging.md](messaging.md#keeping-a-copy)); a tally shows the same in a command's handler:

```csharp
using Dapper;
using PouryaKit.Cqrs;
using PouryaKit.Domain;
using PouryaKit.Persistence;

// Notes/Features/WriteNote/: the note through its repository, its tally in a projection table
internal sealed class WriteNoteCommandHandler(IRepository<Note, NoteId> notes, IUnitOfWorkConnection unitOfWork)
    : CommandHandler<WriteNoteCommand>
{
    public override async Task<Result> Handle(WriteNoteCommand command, CancellationToken cancellationToken)
    {
        await notes.AddAsync(Note.Write(command.Text), cancellationToken);

        var (connection, transaction) = unitOfWork.Join();
        await connection.ExecuteAsync(
            new CommandDefinition(
                """
                insert into notes.note_tallies (text, count) values (@Text, 1)
                on conflict (text) do update set count = note_tallies.count + 1
                """,
                new { command.Text },
                transaction,
                cancellationToken: cancellationToken
            )
        );
        return Result.Success();
    }
}
```

- `Join()` returns the active Unit of Work's connection and transaction. Call it only from a command's handler, a Domain Event Handler, or a Notification or Integration Event Handler, whose delivery runs as a command. It throws `InvalidOperationException` in [code that only reads](commands-queries-validation.md#code-that-only-reads), a query sent from a command's handler included, and wherever no Unit of Work is active.
- Pass the transaction to every statement through `CommandDefinition`. The connection and the transaction belong to the Unit of Work: never dispose, commit or roll back either, and never run transaction control (`begin`, `commit`, `rollback`, `savepoint`) as SQL.
- Write only to the Module's own tables that no aggregate maps. An aggregate changes only through its repository; a plain-SQL write to its table bypasses its version, its audit and soft-delete fields and its Snapshots. A structure test rejects SQL in a Module's code that inserts into, updates, deletes from, merges into or truncates a table any Module's aggregate maps, with or without its schema ([testing.md](testing.md#structure-tests)).
- The statement runs when you call it, inside the transaction, before the Unit of Work saves what the repositories track. It cannot see or reference rows added through a repository in the same Operation.
- If the Operation fails anywhere, the write rolls back with everything else; on a success it commits in the same commit.

## Testing

- Test a read through its Module's Gateway against the hosted application ([testing.md](testing.md#each-test)): send the commands that write the rows, then the query, and assert on its Result, a missing row's `NotFound` included. A Gateway call returns after its commit, so a query sent next reads what the commands wrote. `DapperTests.Get_and_list_read_what_commands_committed_and_a_missing_product_is_not_found` in the Kit's repository is the pattern.
- Test a Page read with rows that its filters, its sort and its page size each tell apart, and assert the items, their order and the total. Then send a request outside the declaration and assert its validation Errors. `PagingTests` proves that the Proving Sample's Dapper and EF Core reads of one declaration answer the same Page, and that SQL in a filter value stays a value.
- Test a plain-SQL write in the Unit of Work with a command that succeeds and one that fails after writing, and read the table back with your own SQL on `Fixture.ConnectionString`: only the first one's row is there, as `DapperTests.A_dapper_write_on_the_unit_of_work_connection_commits_with_the_operation_and_rolls_back_with_it` shows.
