// Jenkinsfile — Opción B: Groovy nativo (sin Docker ni Python)
// Sincroniza los .md de docs/ con Wiki.js usando la API GraphQL directamente.
//
// Mejoras:
//   - Lee la configuración de wikijs-sync.yml (readYaml) en vez de tenerla fija aquí.
//   - Solo publica desde la rama principal (si el job es multibranch).
//   - Quita el H1 del contenido: Wiki.js ya pinta el título y salía duplicado.
//   - Respeta el campo 'estado' de la cabecera: solo 'vigente' se publica.
//   - Convierte los enlaces relativos entre .md en rutas de la wiki.
//   - Detecta cambios también en título y descripción, no solo en el contenido.
//   - Envía el cuerpo en UTF-8 de forma explícita (tildes y eñes).

import groovy.json.JsonOutput
import groovy.json.JsonSlurperClassic

node {
    checkout scm

    if (env.BRANCH_NAME && !(env.BRANCH_NAME in ['main', 'master'])) {
        echo "Rama ${env.BRANCH_NAME}: no se publica en la wiki"
        return
    }

    def yml = readYaml(file: 'wikijs-sync.yml')
    def config = [
        wikiUrl:      'https://pwikijs.aws.umh.es',
        credentialId: 'wikijs-tokengrovy',
        docsDir:      (yml.docs_dir ?: 'docs').toString().replaceAll(/\/+$/, ''),
        basePath:     yml.base_path.toString().replaceAll(/^\/+|\/+$/, ''),
        locale:       (yml.locale ?: 'es').toString()
    ]

    int creadas = 0, actualizadas = 0, sinCambios = 0, errores = 0

    withCredentials([string(credentialsId: config.credentialId, variable: 'WIKI_TOKEN')]) {
        def ficheros = findFiles(glob: "${config.docsDir}/**/*.md")
        echo "Sincronizando ${ficheros.size()} ficheros .md con Wiki.js"
        echo "Base path: ${config.basePath}"

        for (f in ficheros) {
            String rel  = f.path.substring(config.docsDir.length() + 1)
            String ruta = rutaWiki(config.basePath, rel)
            def doc = parsear(readFile(file: f.path, encoding: 'UTF-8'), rel,
                              config.basePath, config.locale)

            try {
                def actual = gql(config.wikiUrl, WIKI_TOKEN,
                    'query($p:String!,$l:String!){ pages { singleByPath(path:$p, locale:$l){ id content title description isPublished } } }',
                    [p: ruta, l: config.locale])
                def pagina = actual?.data?.pages?.singleByPath

                if (!pagina) {
                    def r = gql(config.wikiUrl, WIKI_TOKEN,
                        '''mutation($c:String!,$d:String!,$p:String!,$t:String!,$l:String!,$pub:Boolean!){
                             pages { create(content:$c, description:$d, editor:"markdown",
                                            isPublished:$pub, isPrivate:false, locale:$l,
                                            path:$p, tags:[], title:$t) {
                               responseResult { succeeded message } } } }''',
                        [c: doc.contenido, d: doc.descripcion, p: ruta, t: doc.titulo,
                         l: config.locale, pub: doc.publicar])
                    comprobar(r, 'create')
                    echo "  + crear      ${ruta}${doc.publicar ? '' : ' (sin publicar: ' + doc.estado + ')'}"
                    creadas++
                } else if (normalizar(pagina.content) == normalizar(doc.contenido)
                           && pagina.title == doc.titulo
                           && pagina.description == doc.descripcion
                           && pagina.isPublished == doc.publicar) {
                    sinCambios++
                } else {
                    def r = gql(config.wikiUrl, WIKI_TOKEN,
                        '''mutation($id:Int!,$c:String!,$t:String!,$d:String!,$pub:Boolean!){
                             pages { update(id:$id, content:$c, title:$t, description:$d,
                                            isPublished:$pub) {
                               responseResult { succeeded message } } } }''',
                        [id: pagina.id, c: doc.contenido, t: doc.titulo,
                         d: doc.descripcion, pub: doc.publicar])
                    comprobar(r, 'update')
                    echo "  ~ actualizar ${ruta}"
                    actualizadas++
                }
            } catch (e) {
                echo "  ! error      ${ruta}: ${e.message}"
                errores++
            }
        }
    }

    echo "\nResumen: ${creadas} creadas, ${actualizadas} actualizadas, ${sinCambios} sin cambios, ${errores} errores."
    if (errores > 0) {
        error("Sincronización terminó con ${errores} errores")
    }
}

// ============================================================================
// Funciones auxiliares
// ============================================================================

def gql(String url, String token, String query, Map vars) {
    def resp = httpRequest(
        url: "${url}/graphql",
        httpMode: 'POST',
        contentType: 'APPLICATION_JSON_UTF8',
        customHeaders: [[name: 'Authorization', value: "Bearer ${token}", maskValue: true]],
        requestBody: JsonOutput.toJson([query: query, variables: vars]),
        validResponseCodes: '200',
        quiet: true
    )
    return new JsonSlurperClassic().parseText(resp.content)
}

def comprobar(Map r, String op) {
    if (r.errors) {
        throw new Exception("${op}: ${r.errors[0].message}")
    }
    def res = r.data.pages[op].responseResult
    if (!res.succeeded) {
        throw new Exception("${op}: ${res.message}")
    }
}

@NonCPS
String normalizar(String texto) {
    return (texto ?: '').replaceAll(/\r\n/, '\n').trim()
}

// docs/faq.md -> <base>/faq ; docs/README.md -> <base> ; docs/a/README.md -> <base>/a
@NonCPS
String rutaWiki(String basePath, String rel) {
    String sinExt = rel.replaceAll(/\.md$/, '')
    sinExt = sinExt.replaceAll(/(?i)(^|\/)(?:README|index)$/, '')
    String ruta = [basePath, sinExt].findAll { it }.join('/')
    return ruta.replaceAll(/\/+$/, '')
}

// Resuelve un enlace relativo (p. ej. ../a/b.md) desde el fichero 'rel'
// y devuelve la ruta absoluta de la wiki (/es/<base>/a/b).
@NonCPS
String enlaceWiki(String rel, String destino, String basePath, String locale) {
    List<String> partes = rel.tokenize('/')
    partes = partes.size() > 1 ? partes[0..-2] : []
    for (String p in destino.tokenize('/')) {
        if (p == '..') { if (partes) partes.remove(partes.size() - 1) }
        else if (p != '.') { partes << p }
    }
    return '/' + locale + '/' + rutaWiki(basePath, partes.join('/'))
}

@NonCPS
Map parsear(String texto, String rel, String basePath, String locale) {
    String cuerpo = texto.replaceAll(/\r\n/, '\n')

    // Cabecera YAML: se quita del contenido; 'estado' decide si se publica.
    String estado = 'vigente'
    def fm = (cuerpo =~ /(?s)\A---\n(.*?)\n---\n/)
    if (fm.find()) {
        def est = (fm.group(1) =~ /(?m)^estado:\s*(\S+)/)
        if (est.find()) { estado = est.group(1).toLowerCase() }
        cuerpo = cuerpo.substring(fm.end())
    }

    // Título: primer H1, que se quita del contenido para no duplicarlo.
    String titulo = rel.replaceAll(/\.md$/, '').tokenize('/').last()
    def h1 = (cuerpo =~ /(?m)^#\s+(.+)$/)
    if (h1.find()) {
        titulo = h1.group(1).trim()
        cuerpo = cuerpo.substring(0, h1.start()) + cuerpo.substring(h1.end())
    }

    // Enlaces relativos a otros .md -> rutas de la wiki. Respeta #anclas.
    cuerpo = cuerpo.replaceAll(/\]\((?!https?:|\/|#|mailto:)([^)\s]+?)\.md(#[^)\s]*)?\)/) { all, destino, ancla ->
        '](' + enlaceWiki(rel, destino + '.md', basePath, locale).replaceAll(/\.md$/, '') + (ancla ?: '') + ')'
    }

    cuerpo = cuerpo.trim()
    String descripcion = cuerpo.split(/\n\s*\n/)[0].replaceAll(/\s+/, ' ').take(200)
    if (!descripcion || descripcion.startsWith('|') || descripcion.startsWith('```') || descripcion.startsWith('#')) {
        descripcion = titulo
    }

    return [titulo: titulo, contenido: cuerpo, descripcion: descripcion,
            estado: estado, publicar: (estado == 'vigente')]
}
