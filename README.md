# Guia de API – Métodos HTTP & Model Binding (.NET)

> Este guia define quando usar **GET, POST, PUT, PATCH, DELETE**, como mapear dados com **FromQuery, FromBody, FromForm, FromRoute**, e boas práticas de resposta, paginação e documentação (Swagger).

## Apis Rest - Verbos e Model Binding Api .Net Core

- [Princípios](#princípios)
- [Padrões de Rotas](#padrões-de-rotas)
- [Quando usar cada método](#quando-usar-cada-método)
- [Códigos de status](#códigos-de-status)
- [Model Binding (.NET)](#model-binding-net)
- [Exemplos (Controllers)](#exemplos-controllers)
- [Paginação, ordenação e filtros](#paginação-ordenação-e-filtros)
- [Uploads (multipart/form-data)](#uploads-multipartform-data)
- [Checklist rápido](#checklist-rápido)

---

## Princípios
- **URI representam recursos (plural)**: `/v1/usuarios`, `/v1/usuarios/{id}`.
- **Idempotência**  
  - `GET`, `PUT`, `DELETE`: idempotentes.  
  - `POST`, `PATCH`: não idempotentes (por padrão).
- **Sem efeitos colaterais em GET** (somente leitura).
- **Corpo de resposta consistente** + **erros padronizados**.

## Padrões de Rotas
- Coleções: `GET /recurso`
- Item: `GET /recurso/{id}`
- Criação: `POST /recurso`
- Substituição: `PUT /recurso/{id}`
- Atualização parcial: `PATCH /recurso/{id}`
- Remoção: `DELETE /recurso/{id}`
- Ações não-CRUD: `POST /recurso/{id}:acao` (ex.: `/usuarios/{id}:ativar`) – documente claramente.

## Quando usar cada método

| Método  | Use quando… | Corpo de requisição | Respostas típicas |
|---|---|---|---|
| **GET** `/recurso`, `/recurso/{id}` | Ler lista/um item, sem alterar estado | **Não** | `200 OK`, `206 Partial Content`, `404 Not Found` |
| **POST** `/recurso` | Criar **novo** recurso / executar **ação** não idempotente | **Sim** (JSON) | `201 Created` + `Location`, `202 Accepted`, `400/409` |
| **PUT** `/recurso/{id}` | **Substituir** o recurso inteiro | **Sim** (DTO completo) | `200 OK` ou `204 No Content`, `404` |
| **PATCH** `/recurso/{id}` | **Atualizar parcialmente** campos | **Sim** (merge/patch) | `200 OK` ou `204`, `404` |
| **DELETE** `/recurso/{id}` | Remover recurso | **Não** | `204 No Content`, `404` |

> Regra prática: **PUT = substituição completa**, **PATCH = parcial**, **POST = criar/ações**.

## Códigos de status
- `200 OK` – sucesso com corpo.  
- `201 Created` – em criação; inclua `Location: /v1/recurso/{id}`.  
- `202 Accepted` – processamento assíncrono.  
- `204 No Content` – sucesso sem corpo (PUT/DELETE/PATCH).  
- `400 Bad Request` – validação de entrada.  
- `401 Unauthorized` / `403 Forbidden` – autenticação/autorização.  
- `404 Not Found` – não encontrado.  
- `409 Conflict` – conflito de estado (ex.: duplicidade).  
- `422 Unprocessable Entity` – validações de domínio.  

## Model Binding (.NET)
- **[FromRoute]**: segmentos da rota (ex.: `{id}`).  
- **[FromQuery]**: filtros/paginação/ordenar (parâmetros simples na URL).  
- **[FromBody]**: JSON do corpo (criação/atualização). Apenas **um** `[FromBody]` por ação.  
- **[FromForm]**: `multipart/form-data` (forms e **uploads**).  

### Quando usar
- Busca/paginação: **Query String** → `[FromQuery]`.  
- Criação/atualização JSON: **Body** → `[FromBody]`.  
- Uploads/Forms: **multipart** → `[FromForm]` (ou `Request.Form`).  
- IDs na rota: `[FromRoute]`.

---

## Exemplos (Controllers)

### GET com filtros e paginação
```csharp
[HttpGet]
public async Task<ActionResult<PagedResult<UsuarioDto>>> Get(
    [FromQuery] string? nome,
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 20,
    [FromQuery] string? orderBy = "nome",
    [FromQuery] string? orderDir = "asc")
{
    var r = await _svc.BuscarAsync(nome, page, pageSize, orderBy, orderDir);
    Response.Headers.Append("X-Total-Count", r.Total.ToString());
    return Ok(r);
}
```

### POST (criar) com `201 Created + Location`
```csharp
[HttpPost]
[ProducesResponseType(typeof(UsuarioDto), StatusCodes.Status201Created)]
public async Task<ActionResult<UsuarioDto>> Post([FromBody] UsuarioCreateDto dto)
{
    var u = await _svc.CriarAsync(dto);
    return CreatedAtAction(nameof(GetById), new { id = u.Id }, u);
}

[HttpGet("{id:guid}")]
public async Task<ActionResult<UsuarioDto>> GetById([FromRoute] Guid id)
{
    var u = await _svc.ObterAsync(id);
    return u is null ? NotFound() : Ok(u);
}
```

### PUT (substituição completa)
```csharp
[HttpPut("{id:guid}")]
public async Task<IActionResult> Put([FromRoute] Guid id, [FromBody] UsuarioUpdateDto dto)
{
    var ok = await _svc.SubstituirAsync(id, dto); // valida DTO completo
    return ok ? NoContent() : NotFound();
}
```

### PATCH (parcial) – JSON Merge Patch (`application/merge-patch+json`)
```csharp
[HttpPatch("{id:guid}")]
public async Task<IActionResult> PatchMerge(
    [FromRoute] Guid id,
    [FromBody] JsonDocument mergePatch) // { "nome": "novo" }
{
    var ok = await _svc.MergePatchAsync(id, mergePatch);
    return ok ? NoContent() : NotFound();
}
```

### PATCH – JSON Patch (`application/json-patch+json`)
```csharp
[HttpPatch("{id:guid}")]
public async Task<IActionResult> Patch(
    [FromRoute] Guid id,
    [FromBody] JsonPatchDocument<UsuarioUpdateDto> patch)
{
    var model = await _svc.ObterParaAlteracaoAsync(id);
    if (model is null) return NotFound();

    patch.ApplyTo(model, ModelState);
    if (!ModelState.IsValid) return ValidationProblem(ModelState);

    await _svc.SalvarAsync(id, model);
    return NoContent();
}
```

### DELETE
```csharp
[HttpDelete("{id:guid}")]
public async Task<IActionResult> Delete([FromRoute] Guid id)
{
    var ok = await _svc.RemoverAsync(id);
    return ok ? NoContent() : NotFound();
}
```


---

## Paginação, ordenação e filtros
- **Query**: `?page=1&pageSize=20&orderBy=nome&orderDir=asc&nome=ana`
- **Headers**: `X-Total-Count: <total>`
- **Link Header** (opcional): `Link: <...page=2>; rel="next", <...page=10>; rel="last"`

**Contrato de resposta paginada**
```csharp
public sealed record PagedResult<T>(IReadOnlyList<T> Items, int Page, int PageSize, int Total);
```

---

## Uploads (multipart/form-data)

### DTO com `[FromForm]`
```csharp
public class UploadDocumentoDto
{
    [FromForm] public string Tipo { get; set; } = default!;
    [FromForm] public IFormFile Arquivo { get; set; } = default!;
}

[HttpPost("documentos")]
[RequestSizeLimit(50_000_000)]
public async Task<IActionResult> Upload([FromForm] UploadDocumentoDto dto)
{
    using var stream = dto.Arquivo.OpenReadStream();
    await _docService.SalvarAsync(dto.Tipo, dto.Arquivo.FileName, stream);
    return Accepted();
}
```

### Acesso direto ao **Request.Form** (vários arquivos/campos dinâmicos)
```csharp
[HttpPost("documentos/form")]
public async Task<IActionResult> UploadForm()
{
    var form = await Request.ReadFormAsync();
    var tipo = form["tipo"].ToString();
    foreach (var file in form.Files)
        await _docService.SalvarAsync(tipo, file.FileName, file.OpenReadStream());
    return Accepted();
}
```

---


## Resumo 
- [ ] Método correto (POST criar / PUT substituir / PATCH parcial / DELETE excluir / GET ler).  
- [ ] Idempotência respeitada.  
- [ ] `201 + Location` em criações.  
- [ ] `204` quando não houver corpo a retornar.  
- [ ] Filtros/paginação via **Query**, corpo em **Body**, uploads via **Form**.  
- [ ] DTOs validados (DataAnnotations/FluentValidation).  
- [ ] Erros padronizados (ProblemDetails).  
- [ ] Swagger com exemplos e content-types corretos.
