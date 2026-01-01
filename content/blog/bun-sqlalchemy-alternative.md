+++
title = "Go's Bun ORM - alternative to Python's SQLAlchemy"
date = 2026-01-02
draft = false
+++

Initially, my [Greener](https://github.com/cephei8/greener) project was implemented in Python.<br />
When I decided to switch to Go, I started looking for Go ORM library.
In Python the de facto standard is [SQLAlchemy](https://www.sqlalchemy.org).
I liked SQLAlchemy and was hoping to find something similar for Go.

Specifically, I was looking for the following:

1.  Actively maintained
2.  Support for different databases (in the sense that same code works with different databases)
3.  ORM layer (for simple queries)
4.  SQL eDSL (for complex queries)
5.  Support for dynamic queries (e.g. variable set of query conditions etc.)
6.  Migration system

I settled on [Bun](https://bun.uptrace.dev) - while not being as ergonomic as SQLAlchemy
(not Bun's fault, I think it's done a great job given the programming language)
and not ticking all the boxes, it still covered the most important parts.


## Support for different databases {#support-for-different-databases}

The databases I wanted were SQLite, PostgreSQL and MySQL.<br />
Bun supports all of them (and more), so I didn't look any further.


## ORM layer {#orm-layer}

Defining models and using them for queries is simple and straightforward.

The following code examples demonstrate common operations in Python's SQLAlchemy and Go's Bun.


### Code examples {#code-examples}


#### Defining models {#defining-models}

Python:

```python
class APIKey(Base):
    __tablename__ = "apikeys"

    id: Mapped[UUID] = mapped_column(default=uuid4, primary_key=True)
    description: Mapped[str | None] = mapped_column(Text)
    secret_salt: Mapped[bytes]
    secret_hash: Mapped[bytes]
    created_at: Mapped[datetime.datetime] = mapped_column(
        DateTimeUTC(timezone=True),
        default=lambda: datetime.datetime.now(datetime.timezone.utc),
    )

    user_id: Mapped[UUID] = mapped_column(ForeignKey("users.id"))
    user: Mapped[User] = relationship(lazy="noload")
```

Go:

```go
type APIKey struct {
	bun.BaseModel `bun:"table:apikeys"`

	ID          BinaryUUID `bun:"id,notnull"`
	Description *string    `bun:"description"`
	SecretSalt  []byte     `bun:"secret_salt,notnull"`
	SecretHash  []byte     `bun:"secret_hash,notnull"`
	CreatedAt   time.Time  `bun:"created_at,nullzero,notnull"`
	UserID      BinaryUUID `bun:"user_id,notnull"`
}
```


#### Inserting data {#inserting-data}

Python:

```python
api_key = APIKey(
    description="My API key",
    secret_salt=password_salt,
    secret_hash=password_hash,
    user_id=user.id,
)

async_session_maker = async_sessionmaker(bind=db_engine)
async with async_session_maker() as db_session:
    db_session.add(api_key)
    await db_session.commit()
```

Go:

```go
apiKey := &APIKey{
	ID:          BinaryUUID(uuid.New()),
    Description: description,
	SecretSalt:  salt,
	SecretHash:  secretHash,
	CreatedAt:   time.Now(),
	UserID:      BinaryUUID(userId),
}

_, err := db.NewInsert().Model(apiKey).Exec(ctx)
if err != nil {
    return err
}
```


#### Fetching data {#fetching-data}

Python:

```python
async_session_maker = async_sessionmaker(bind=db_engine)
async with async_session_maker() as db_session:
    stmt = (
        select(APIKey)
        .where(APIKey.user_id == user_id)
        .order_by(desc(APIKey.created_at))
    )
    result = await db_session.execute(stmt)
    api_keys = result.scalars().all()
```

Go:

```go
var apiKeys []APIKey

err := db.NewSelect().
	Model(&apiKeys).
	Where("? = ?", bun.Ident("user_id"), BinaryUUID(userId)).
	OrderBy("created_at", bun.OrderDesc).
	Scan(ctx)

if err != nil {
    return err
}
```


### Mapping types to database types {#mapping-types-to-database-types}

Notice that my Go code uses \`BinaryUUID\` - it's needed to map UUID to binary on SQLite and MySQL.
It can be implemented as follows (\`Dialect\` must be set beforehand e.g. on program start):

Go:

```go
var Dialect schema.Dialect

type BinaryUUID uuid.UUID

func (u *BinaryUUID) Scan(value any) error {
	if value == nil {
		return fmt.Errorf("cannot scan nil into BinaryUUID")
	}

	switch Dialect.(type) {
	case *pgdialect.Dialect:
		switch v := value.(type) {
		case string:
			parsed, err := uuid.Parse(v)
			if err != nil {
				return fmt.Errorf("failed to parse UUID string: %w", err)
			}
			*u = BinaryUUID(parsed)
			return nil
		case []byte:
			parsed, err := uuid.ParseBytes(v)
			if err != nil {
				return fmt.Errorf("failed to parse UUID bytes: %w", err)
			}
			*u = BinaryUUID(parsed)
			return nil
		default:
			return fmt.Errorf("unsupported type for PostgreSQL UUID: %T", value)
		}
	case *mysqldialect.Dialect, *sqlitedialect.Dialect:
		bytes, ok := value.([]byte)
		if !ok {
			return fmt.Errorf("expected []byte for MySQL/SQLite UUID, got %T", value)
		}
		if len(bytes) != 16 {
			return fmt.Errorf("expected 16 bytes for UUID, got %d", len(bytes))
		}
		parsed, err := uuid.FromBytes(bytes)
		if err != nil {
			return fmt.Errorf("failed to parse UUID from bytes: %w", err)
		}
		*u = BinaryUUID(parsed)
		return nil
	default:
		return fmt.Errorf("unknown dialect type: %T", Dialect)
	}
}

func (u BinaryUUID) Value() (driver.Value, error) {
	id := uuid.UUID(u)

	switch Dialect.(type) {
	case *pgdialect.Dialect:
		return id.String(), nil
	case *mysqldialect.Dialect, *sqlitedialect.Dialect:
		bytes, err := id.MarshalBinary()
		if err != nil {
			return nil, fmt.Errorf("failed to marshal UUID to binary: %w", err)
		}
		return bytes, nil
	default:
		return nil, fmt.Errorf("unknown dialect type: %T", Dialect)
	}
}

func (u BinaryUUID) String() string {
	return uuid.UUID(u).String()
}

func (u BinaryUUID) UUID() uuid.UUID {
	return uuid.UUID(u)
}
```

SQLAlchemy also allows customizations, but I changed the UUID mapping to binary after the Go rewrite,
so I don't have a hands-on example in Python.


## SQL eDSL {#sql-edsl}

Greener supports query language that lets users do filtering and grouping.<br />
That means that the underlying SQL query will have variable set of conditions and variable set of columns to group by.

The queries that I needed to implement were not particularly complex, but they also needed the following:

-   CTEs (Common Table Expressions)
-   Window functions

The following code snippets demonstrate building such a dynamic query in SQLAlchemy vs Bun.<br />
Although the Go version may be slightly different than the Python version, the code snippets show how each framework "feels".

Python:

```python
select_columns = [
    c.label(la) for c, la in zip(group_columns, group_column_labels)
]
cte_query = select(
    *select_columns, func.min(Testcase.status).label("aggregated_status")
).select_from(Testcase)

for token, tbl in zip(query_ast.group_by.tokens, group_tables):
    match token.token_type:
        case GroupByTokenType.SESSION_ID:
            cte_query = cte_query.join(tbl, Testcase.session_id == tbl.id)
        case GroupByTokenType.TAG:
            cte_query = cte_query.join(
                tbl,
                and_(
                    Testcase.session_id == tbl.session_id,
                    tbl.key == token.value,
                ),
            )
        case _:
            assert_never(token.token_type)

cte_query = (
    cte_query.where(and_(Testcase.user_id == request.user.id))
    .group_by(*group_columns)
    .order_by(*group_columns)
)

where_cond = build_query_conditions(query_ast.main_query)
if where_cond is not None:
    cte_query = cte_query.where(where_cond)

cte = cte_query.cte("cte")
query = (
    select(
        cte,
        func.count(1).over().label("total_count"),
    )
    .select_from(cte)
    .offset(offset)
    .limit(limit)
)
```

Go:

```go
groupCols := []string{}
orderCols := []string{}

cteQuery := db.NewSelect().Table(fmt.Sprintf("%s", testcasesTable))

labelJoinIdx := 0
for _, token := range groupBy.Tokens {
	switch t := token.(type) {
	case query.SessionGroupToken:
		idCol := fmt.Sprintf("%s.id", sessionsTable)
		cteQuery = cteQuery.ColumnExpr(
			"? AS ?", bun.Ident(idCol),
			bun.Ident("session_id"),
		)
		groupCols = append(groupCols, idCol)
		orderCols = append(orderCols, idCol)

	case query.TagGroupToken:
		alias := fmt.Sprintf("l%d", labelJoinIdx)
		valCol := fmt.Sprintf("%s.value", alias)
		cteQuery = cteQuery.ColumnExpr(
			"? AS ?",
			bun.Ident(valCol),
			bun.Ident(fmt.Sprintf("\"%s\"", t.Tag)),
		)
		groupCols = append(groupCols, valCol)
		orderCols = append(orderCols, valCol)
		labelJoinIdx++
	}
}

cteQuery = cteQuery.ColumnExpr(
	"MIN(?) AS ?",
	bun.Ident(fmt.Sprintf("%s.status", testcasesTable)),
	bun.Ident("aggregated_status"),
)
cteQuery = cteQuery.ColumnExpr(
	"COUNT(DISTINCT ?) AS ?",
	bun.Ident(fmt.Sprintf("%s.id", testcasesTable)),
	bun.Ident("testcase_count"),
)

labelJoinIdx = 0
for _, token := range groupBy.Tokens {
	switch t := token.(type) {
	case query.SessionGroupToken:
		cteQuery = cteQuery.Join(
			"JOIN ? ON ? = ?",
			bun.Ident(fmt.Sprintf("%s", sessionsTable)),
			bun.Ident(fmt.Sprintf("%s.session_id", testcasesTable)),
			bun.Ident(fmt.Sprintf("%s.id", sessionsTable)),
		)

	case query.TagGroupToken:
		alias := fmt.Sprintf("l%d", labelJoinIdx)
		cteQuery = cteQuery.Join(
			"JOIN ? AS ? ON ? = ? AND ? = ?",
			bun.Ident(fmt.Sprintf("%s", labelsTable)),
			bun.Ident(alias),
			bun.Ident(fmt.Sprintf("%s.session_id", testcasesTable)),
			bun.Ident(fmt.Sprintf("%s.session_id", alias)),
			bun.Ident(fmt.Sprintf("%s.key", alias)),
			t.Tag,
		)
		labelJoinIdx++
	}
}

cteQuery = cteQuery.Where(
	"? = ?",
	bun.Ident(fmt.Sprintf("%s.user_id", testcasesTable)),
	userID,
)
cteQuery = applySelectQuery(cteQuery, queryAST.SelectQuery)

for _, col := range groupCols {
	cteQuery = cteQuery.Group(col)
}
for _, col := range orderCols {
	cteQuery = cteQuery.Order(col)
}

mainQuery := db.NewSelect().
	Table("cte").
	Column("*").
	ColumnExpr("COUNT(?) OVER() AS ?", 1, bun.Ident("total_count")).
	With("cte", cteQuery)

mainQuery, err := applyOffsetLimit(mainQuery, queryAST)
if err != nil {
	return nil, err
}
```

Overall, Bun's eDSL is capable enough for more complex queries.


## Migration system {#migration-system}

SQLAlchemy together with [Alembic](https://alembic.sqlalchemy.org) make a great database migration tool.<br />
It provides many features (Alembic's goals are stated in its [GitHub repository](https://github.com/sqlalchemy/alembic)),
but the main one for me was the eDSL to have a single migration work with different databases.

Unfortunately, Bun doesn't help here.<br />
There's eDSL, but it's required to write database-specific code (e.g. for database-specific types),
whereas in Alembic you work with SQLAlchemy types, which then are automatically mapped to database types.

I ended up using [Dbmate](https://github.com/amacneil/dbmate) with raw SQL migrations for each supported database.<br />
It's possible to do migrations in Bun, but for my use case there're no benefits using it comparing to raw SQL.


## Conclusion {#conclusion}

Bun is a solid choice for a Go ORM with SQL eDSL support.

It supports SQLite, PostreSQL, MySQL and more.<br />
ORM layer is easy to use, and eDSL can be used for more complex queries (supporting subqueries, CTEs, window functions).<br />
The same application code (ORM layer or eDSL) can be used with different databases (e.g. depending on the database provided in runtime).
