project(papers)

cmake_minimum_required(VERSION 3.2)

# This script uses pandoc to convert from commonmark (*.md) files to HTML (*.html) files.
find_program(PANDOC_EXECUTABLE pandoc)
if (PANDOC_EXECUTABLE STREQUAL PANDOC_EXECUTABLE-NOTFOUND)
    message(FATAL_ERROR "Cannot find pandoc executable.")
endif ()

file(GLOB_RECURSE PAPERS
        LIST_DIRECTORIES false
        RELATIVE "${CMAKE_CURRENT_LIST_DIR}"
        "*/*.md")

# Given project-relative path to commonmark file,
# converts to HTML file and outputs to htdocs directory.
function(md_to_html MD_FILE FILTER)
    get_filename_component(FILENAME ${MD_FILE} NAME_WE)
    get_filename_component(MD_PATH ${MD_FILE} DIRECTORY)

    set(HTML_FILE "${FILENAME}.html")
    set(HTML_PATH "${CMAKE_CURRENT_LIST_DIR}/htdocs/${MD_PATH}")

    string(REPLACE "/" "-" TARGET "${MD_PATH}/${FILENAME}")

    add_custom_target(
            "${TARGET}" ALL
            COMMAND mkdir -p "${HTML_PATH}"
            COMMAND sed ${FILTER} "${CMAKE_CURRENT_LIST_DIR}/${MD_FILE}" | ${PANDOC_EXECUTABLE} --from=commonmark --to=html --standalone --include-in-header=${CMAKE_CURRENT_LIST_DIR}/css.xml -o "${HTML_PATH}/${HTML_FILE}"
    )
endfunction(md_to_html)

foreach (PAPER ${PAPERS})
    md_to_html(${PAPER} "''")
endforeach()

md_to_html(README.md "'s/.md/.html/g'")
