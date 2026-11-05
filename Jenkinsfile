// Jenkinsfile — Opción B: Groovy nativo (sin Docker ni Python)
// Sincroniza los .md de docs/ con Wiki.js usando la API GraphQL directamente.

import groovy.json.JsonOutput
import groovy.json.JsonSlurperClassic

node {
    checkout scm

    // Configuración
    def config = [
        wikiUrl:      'https://pwikijs.aws.umh.es',
        credentialId: 'wikijs-tokengrovy',
        docsDir:      'docs',
        basePath:     'APLICACIONES/prueba-jenkins',
        locale:       'es'
    ]

    int creadas = 0, actualizadas = 0, sinCambios = 0, errores = 0

    withCredentials([string(credentialsId: config.credentialId, variable: 'WIKI_TOKEN')]) {
        def ficheros = findFiles(glob: "${config.docsDir}/**/*.md")
        
        echo "Sincronizando ${ficheros.size()} ficheros .md con Wiki.js"
        echo "Base path: ${config.basePath}"

        for (f in ficheros) {
            String rel = f.path.substring(config.docsDir.length() + 1)
            String ruta = rutaWiki(config.basePath, rel)
            def doc = parsear(readFile(file: f.path, encoding: 'UTF-8'), rel)

            try {
                // Buscar si la página existe
                def actual = gql(config.wikiUrl, WIKI_TOKEN,
                    'query($p:String!,$l:String!){ pages { singleByPath(path:$p, locale:$l){ id content title } } }',
                    [p: ruta, l: config.locale])
                def pagina = actual?.data?.pages?.singleByPath

                if (!pagina) {
                    // Crear página nueva
                    def r = gql(config.wikiUrl, WIKI_TOKEN,
                        '''mutation($c:String!,$d:String!,$p:String!,$t:String!,$l:String!,$pub:Boolean!){
                             pages { create(content:$c, description:$d, editor:"markdown",
                                            isPublished:$pub, isPrivate:false, locale:$l,
                                            path:$p, tags:[], title:$t) {
                               responseResult { succeeded message } } } }''',
                        [c: doc.contenido, d: doc.descripcion, p: ruta, t: doc.titulo,
                         l: config.locale, pub: true])
                    comprobar(r, 'create', ruta)
                    echo "  + crear      ${ruta}"
                    creadas++
                } else if (normalizar(pagina.content) == normalizar(doc.contenido)) {
                    sinCambios++
                } else {
                    // Actualizar página existente
                    def r = gql(config.wikiUrl, WIKI_TOKEN,
                        '''mutation($id:Int!,$c:String!,$t:String!,$d:String!){
                             pages { update(id:$id, content:$c, title:$t, description:$d,
                                            isPublished:true) {
                               responseResult { succeeded message } } } }''',
                        [id: pagina.id, c: doc.contenido, t: doc.titulo, d: doc.descripcion])
                    comprobar(r, 'update', ruta)
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
    def body = JsonOutput.toJson([query: query, variables: vars])
    def resp = httpRequest(
        url: "${url}/graphql",
        httpMode: 'POST',
        contentType: 'APPLICATION_JSON',
        customHeaders: [[name: 'Authorization', value: "Bearer ${token}", maskValue: true]],
        requestBody: body,
        validResponseCodes: '200',
        quiet: true
    )
    return new JsonSlurperClassic().parseText(resp.content)
}

def comprobar(Map r, String op, String ruta) {
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

@NonCPS
String rutaWiki(String basePath, String rel) {
    String sinExt = rel.replaceAll(/\.md$/, '')
    sinExt = sinExt.replaceAll(/(?i)(^|\/)(?:README|index)$/, '')
    String ruta = [basePath, sinExt].findAll { it }.join('/')
    return ruta.replaceAll(/\/+$/, '')
}

@NonCPS
Map parsear(String texto, String rel) {
    String cuerpo = texto.replaceAll(/\r\n/, '\n')

    def fm = (cuerpo =~ /(?s)\A---\n(.*?)\n---\n/)
    if (fm.find()) {
        cuerpo = cuerpo.substring(fm.end())
    }

    String titulo = rel.replaceAll(/\.md$/, '').tokenize('/').last()
    def h1 = (cuerpo =~ /(?m)^#\s+(.+)$/)
    if (h1.find()) {
        titulo = h1.group(1).trim()
    }

    String descripcion = cuerpo.trim().split(/\n\s*\n/)[0]
        .replaceAll(/\s+/, ' ').take(200)
    if (descripcion.startsWith('|') || descripcion.startsWith('```') || descripcion.startsWith('#')) {
        descripcion = titulo
    }

    return [titulo: titulo, contenido: cuerpo.trim(), descripcion: descripcion]
}
