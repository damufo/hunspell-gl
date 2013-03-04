# coding=utf-8

import mmap, os, re, subprocess

#---# Valores predeterminados #----------------------------------------------------------------------------------------#

defaultAff  = u'norma,unidades'
defaultDic  = u'iso639,iso4217,unidades,volga'
defaultCode = u'gl'

#---# Axuda #----------------------------------------------------------------------------------------------------------#

Help("""
Execute «scons aff=<módulos de regras> dic=<módulos de dicionario> code=<código de lingua>» para construír o corrector cos módulos indicados para as regras e o dicionario, empregando o código de lingua indicado para o nome dos ficheiros.

Para combinar varios módulos, sepáreos con comas (sen espazos). Por exemplo:

    scons dic=volga,unidades

Os valores por omisión son:

    Regras: {aff}.
    Dicionario: {dic}.
    Código de lingua: {code} (dá lugar a «{code}.aff» e «{code}.dic»).

Módulos dispoñíbeis:


    REGRAS

    norma           Normas ortográficas e morfolóxicas do idioma galego
                    Real Academia Galega / Instituto da Lingua Galega, 2003.
                    http://www.realacademiagalega.org/PlainRAG/catalog/publications/files/normas_galego05.pdf

    trasno          Flexións especiais para os acordos terminolóxicos do Proxecto Trasno.
                    http://trasno.net/content/resultados-das-trasnadas
            
    unidades        Prefixos e sufixos para símbolos de unidades.
                    http://en.wikipedia.org/wiki/International_System_of_Units
                    Nota: inclúense prefixos para unidades binarias.


    DICIONARIO

    iso639          Códigos de linguas (ISO 639).
                    http://gl.wikipedia.org/wiki/ISO_639

    iso4217         Códigos de moedas (ISO 4217).
                    http://gl.wikipedia.org/wiki/ISO_4217

    trasno          Acordos terminolóxicos do Proxecto Trasno.
                    http://trasno.net/content/resultados-das-trasnadas
            
    unidades        Símbolos de unidades.
                    http://en.wikipedia.org/wiki/International_System_of_Units
                    Nota: inclúense unidades de fóra do S.I., como byte (B) ou quintal métrico (q) ou tonelada (t).

    volga           Vocabulario ortográfico da lingua galega
                    Santamarina Fernández, Antón e González González, Manuel (coord.)
                    Real Academia Galega / Instituto da Lingua Galega, 2004.
                    http://www.realacademiagalega.org/volga/
    
""".format(aff=defaultAff, dic=defaultDic, code=defaultCode))


#---# Builders #-------------------------------------------------------------------------------------------------------#


def initialize(target):
    """ Crea un ficheiro baleiro na ruta indicada.
    """
    with open(target, 'w') as file:
        file.write('')


def isNotUseless(line):
    """ Determina se a liña indicada ten algunha utilidade para o corrector, ou se pola contra se trata dunha liña que
        ten unicamente un comentario, ou se trata duña liña baleira.
    """
    if line[0] == "#":
        return False
    elif line.strip() == "":
        return False
    else:
        return True


def removeAnyCommentFrom(line):
    """ Se a liña ten un comentario, elimínao.

        Nota: se a liña acababa en salto de liña, este tamén se eliminará xunto co comentario.
    """
    index = line.find('#')
    if index < 0:
        return line
    else:
        return line[0:index]


def removeMultipleSpacesOrTabsFrom(line):
    """ Converte as tabulacións da liña en espazos, e reduce estes ao mínimo necesario. É dicir, toda tabulación ou
        grupo de tabulacións e grupo de espazos quedarán reducidos a un único espazo cada un.

        Por exemplo, se a liña é:

            "\t\t\t1\tasdfasdf    2  \t\t  \t 3"

        Devolverase:

            " 1 2 3"
    """
    line = line.replace('\t', ' ')
    line = re.sub(' +', ' ', line)
    return line



def stripLine(line):
    line = removeAnyCommentFrom(line)
    line = line.rstrip() + '\n'
    line = removeMultipleSpacesOrTabsFrom(line)
    return line



def extendAff(targetFilename, sourceFilename):
    """ Inclúe datos do módulo de orixe nun ficheiro .aff, o de destino, que pode que exista ou que non.
    """
    with open(targetFilename, 'a') as targetFile:
        try:
            parsedContent = ""
            with open(sourceFilename) as sourceFile:
                for line in sourceFile:
                    if isNotUseless(line):
                        parsedContent += stripLine(line)
            targetFile.write(parsedContent)
        except IOError:
            pass


def createAff(targetFilename, sourceFilenames):
    """ Constrúe o ficheiro .aff a partir dos módulos indicados.
    """
    initialize(targetFilename)
    for sourceFilename in sourceFilenames:
        extendAff(targetFilename, sourceFilename)


def getParsedDicContent(sourceFilename):
    """ Inclúe datos do módulo de orixe nun ficheiro .dic, o de destino, que pode que exista ou que non.
    """
    try:
        parsedContent = ""
        with open(sourceFilename) as sourceFile:
            for line in sourceFile:
                if isNotUseless(line):
                    parsedContent += stripLine(line)
        return parsedContent
    except IOError:
        return ''


def createDic(targetFilename, sourceFilenames):
    """ Constrúe o ficheiro .dic a partir dos módulos indicados.
    """
    content = ''
    for sourceFilename in sourceFilenames:
        content += getParsedDicContent(sourceFilename)

    linesSeen = set()
    for line in iter(content.splitlines()):
        if line not in linesSeen:
            linesSeen.add(line)

    contentWithoutDuplicates = ""
    for line in sorted(linesSeen):
        contentWithoutDuplicates += line + '\n'

    with open(targetFilename, 'w') as targetFile:
        targetFile.write('{}\n{}'.format(len(linesSeen), contentWithoutDuplicates))


def getAliasFilepathFor(filepath):
    return filepath[:-4] + '_alias' + filepath[-4:]


def applyMakealias(aff, dic):
    """ Executa a orde «makealias», de Hunspell, para reducir o tamaño final dos ficheiros considerablemente.
    """
    affAlias = getAliasFilepathFor(aff)
    dicAlias = getAliasFilepathFor(dic)
    command = 'makealias {dic} {aff}'.format(aff=aff[6:], dic=dic[6:])
    p = subprocess.Popen(command, stdout=subprocess.PIPE, shell=True, cwd=os.path.join(os.getcwd(), 'build'))
    out, err = p.communicate()
    for filepath in [aff, dic]:
        os.remove(filepath)
        os.rename(getAliasFilepathFor(filepath), filepath)


def getModuleListFromModulesString(modulesString):
    modules = []
    if ',' in modulesString:
        modules = ['src/{}'.format(module) for module in modulesString.split(',')]
    else:
        modules = ['src/{}'.format(modulesString)]
    return modules


def getSourceFilesFromModulesStringAndExtension(modulesString, extension):
    sourceFiles = []
    modules = getModuleListFromModulesString(modulesString)
    for module in modules:
        filepath = os.path.join(module, 'main{extension}'.format(extension=extension))
        if os.path.isfile(filepath):
            sourceFiles.append(File(filepath))
    return sourceFiles


def getSourceFiles(dictionary, rules):
    sourceFiles = []
    sourceFiles.extend(getSourceFilesFromModulesStringAndExtension(rules, '.aff'))
    sourceFiles.extend(getSourceFilesFromModulesStringAndExtension(dictionary, '.dic'))
    return sourceFiles


def getFilenamesFromFileEntriesWithMatchingExtensions(fileEntries, extensionList):
    """ Note: Only supports three-letter extensions. For example: '.aff', '.dic', '.rep'.
    """
    filenames = []
    for fileEntry in fileEntries:
        filename = unicode(fileEntry)
        if filename[-4:] in extensionList:
            filenames.append(filename)
    return filenames


def createSpellchecker(target, source, env):
    aff = unicode(target[0])
    dic = unicode(target[1])
    createAff(aff, getFilenamesFromFileEntriesWithMatchingExtensions(source, ['.aff']))
    createDic(dic, getFilenamesFromFileEntriesWithMatchingExtensions(source, ['.dic']))
    applyMakealias(aff, dic)


#---# Análise dos argumentos da chamada #------------------------------------------------------------------------------#

languageCode = ARGUMENTS.get('code', defaultCode)
rules = ARGUMENTS.get('aff', defaultAff)
dictionary = ARGUMENTS.get('dic', defaultDic)


# Construtor para os ficheiros de Hunspell.
env = Environment()
env['BUILDERS']['spellchecker'] = Builder(action = createSpellchecker)


#---# Construción #----------------------------------------------------------------------------------------------------#

env.spellchecker(
    ['build/{}.aff'.format(languageCode), 'build/{}.dic'.format(languageCode)],
    getSourceFiles(dictionary=dictionary, rules=rules)
)

