# streamlit-searchbox

- [streamlit-searchbox](#streamlit-searchbox)
  - [Installation](#installation)
  - [Overview](#overview)
  - [Parameters](#parameters)
    - [Required](#required)
    - [Visual](#visual)
    - [Defaults](#defaults)
    - [Reruns](#reruns)
    - [Transitions](#transitions)
    - [Custom Styles](#custom-styles)
  - [Example](#example)
  - [Styling](#styling)
  - [Contributions](#contributions)
    - [Contributors](#contributors)

---

A streamlit custom component providing a searchbox with autocomplete.

![Example](./assets/example.gif)


## Installation

```python
pip install streamlit-searchbox
```

## Overview

Create a searchbox component and pass a `search_function` that accepts a `str` searchterm. The searchbox is triggered on user input, calls the search function for new options and redraws the page via `st.experimental_rerun()`.

You can either pass a list of arguments, e.g.

```python
import wikipedia
from streamlit_searchbox import st_searchbox

# function with list of labels
def search_wikipedia(searchterm: str) -> List[any]:
    return wikipedia.search(searchterm) if searchterm else []


# pass search function to searchbox
selected_value = st_searchbox(
    search_wikipedia,
    key="wiki_searchbox",
)
```

This example will call the Wikipedia Api to reload suggestions. The `selected_value` will be one of the items the `search_wikipedia` function returns, the suggestions shown in the UI components are a `str` representation. In case you want to provide custom text for suggestions, pass a `Tuple`.

```python
def search(searchterm: str) -> List[Tuple[str, any]]:
    ...
```

## Parameters

To customize the searchbox you can pass the following arguments:


### Required

```python
search_function: Callable[[str], List[any]]
```

Function that will be called on user input

```python
key: str = "searchbox"
```

Streamlit key for unique component identification.

---

### Visual

```python
placeholder: str = "Search ..."
```

Placeholder for empty searches shown within the component.

```python
label: str = None
```

Label shown above the component.

---

### Defaults


```python
default: any = None
```

Default return value in case nothing was submitted or the searchbox cleared.

```python
default_use_searchterm: bool = False
```

Use the current searchterm as a default return value.

```python
default_options: list[str] | None = None
```

Default options that will be shown when first clicking on the searchbox.

### Reruns

```python
rerun_on_update: bool = True
```

Use `st.experimental_rerun()` to reload the app after user input and load new search suggestions. Disabling leads to delay in showing the proper search results.

```python
rerun_scope: Literal["app", "fragment"] = "app",
```

If the rerun should affect the whole app or just the fragment.

```python
debounce: int = 0
```

Delay executing the callback from the react component by `x` milliseconds to avoid too many / redudant requests, i.e. during fast typing.

```python
min_execution_time: int = 0
```

Delay execution after the search function finished to reach a minimum amount of `x` milliseconds. This can be used to avoid fast consecutive reruns, which can cause resets of the component in some streamlit versions `>=1.35`.

---

### Transitions

```python
clear_on_submit: bool = False
```

Automatically clear the input after selection.

```python
edit_after_submit: Literal["disabled", "current", "option", "concat"] = "disabled"
```

Specify behavior for search query after an option is selected.

```python
reset_function: Callable[[], None] | None = None
```

Function that will be called when the combobox is reset.

---

### Custom Styles

```python
style_overrides: dict | None = None
```

See [section](#styling) below for more details.

```python
style_absolute: bool = False
```

Will position the searchbox as an absolute element. *NOTE:* this will affect all searchbox instances and should either be set for all boxes or none. See [#46](https://github.com/m-wrzr/streamlit-searchbox/issues/46) for inital workaround by [@JoshElgar](https://github.com/JoshElgar).

## Example

An example Streamlit app can be found [here](./example.py)

## Styling

To further customize the styling of the searchbox, you can override the default styling by passing `style_overrides` which will be directly applied in the react components. See below for an example, for more information on the available attributes, please see [styling.tsx](./streamlit_searchbox/frontend/src/styling.tsx) as well as the [react-select](https://react-select.com/styles) documentation.

```json
{
   "clear":{
      "width":20,
      "height":20,
      "icon":"cross",
      "clearable":"always"
   },
   "dropdown":{
      "rotate":true,
      "width":30,
      "height":30,
      "fill":"red"
   },
   "searchbox":{
      "menuList":{
         "backgroundColor":"transparent"
      },
      "singleValue":{
         "color":"red"
      },
      "option":{
         "color":"blue",
         "backgroundColor":"yellow"
      }
   }
}
```

## Contributions

We welcome contributions from everyone. Here are a few ways you can help:

- **Reporting bugs**: If you find a bug, please report it by opening an issue.
- **Suggesting enhancements**: If you have ideas on how to improve the project, please share them by opening an issue.
- **Pull requests**: If you want to contribute directly to the code base, please make a pull request. Here's how you can do so:
  1. Fork the repository.
  2. Create a new branch (`git checkout -b feature-branch`).
  3. Make your changes.
  4. Commit your changes (`git commit -am 'Add some feature'`).
  5. Push to the branch (`git push origin feature-branch`).
  6. Create a new Pull Request.

### Contributors

- [@JoshElgar](https://github.com/JoshElgar) absolute positioning workaround
- [@dopc](https://github.com/dopc) bugfix for [#15](https://github.com/m-wrzr/streamlit-searchbox/issues/15)
- [@Jumitti](https://github.com/Jumitti) `st.rerun` compatibility
- [@salmanrazzaq-94](https://github.com/salmanrazzaq-94) `st.fragment` support
- [@hoggatt](https://github.com/hoggatt) `reset_function`



```
adicionar no futuro, fazer um edit no deles 

"""
Módulo para o componente searchbox do Streamlit com suporte a reset automático de estado.
"""

from __future__ import annotations

import datetime
import functools
import logging
import os
import time
from typing import Any, Callable, List, Literal, TypedDict

import streamlit as st
import streamlit.components.v1 as components

try:
    from streamlit import rerun  # type: ignore
except ImportError:
    # Import condicional para versões do Streamlit <1.27
    from streamlit import experimental_rerun as rerun  # type: ignore

# Milissegundos padrão para a função de busca ser executada, usado para evitar
# reruns rápidos consecutivos. Pode ser removido em versões futuras.
# Veja: https://github.com/streamlit/streamlit/issues/9002
MIN_EXECUTION_TIME_DEFAULT = 250 if st.__version__ >= "1.35" else 0

# Diretório de build do componente React
parent_dir = os.path.dirname(os.path.abspath(__file__))
build_dir = os.path.join(parent_dir, "frontend/build")
_get_react_component = components.declare_component(
    "searchbox",
    path=build_dir,
)

logger = logging.getLogger(__name__)


def wrap_inactive_session(func):
    """
    O session_state pode não estar disponível devido a reruns (a chave de estado não pode estar vazia)
    Se o proxy estiver faltando, este thread não está realmente ativo e um retorno antecipado é nulo.
    """

    @functools.wraps(func)
    def inner_function(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except KeyError as error:
            if kwargs.get("key", None) == error.args[0]:
                logger.debug(f"Session Proxy indisponível para a chave: {error.args[0]}")
                return

            raise error

    return inner_function


def _list_to_options_py(options: list[Any] | list[tuple[str, Any]]) -> list[Any]:
    """
    Desempacota as opções de busca para tipos de retorno Python apropriados.
    """
    return [v[1] if isinstance(v, tuple) else v for v in options]


def _list_to_options_js(
    options: list[Any] | list[tuple[str, Any]]
) -> list[dict[str, Any]]:
    """
    Desempacota as opções de busca para uso no componente React.
    """
    return [
        {
            "label": str(v[0]) if isinstance(v, tuple) else str(v),
            "value": i,
        }
        for i, v in enumerate(options)
    ]


def _process_search(
    search_function: Callable[[str], List[Any]],
    key: str,
    searchterm: str,
    rerun_on_update: bool,
    rerun_scope: Literal["app", "fragment"] = "app",
    min_execution_time: int = 0,
    **kwargs,
) -> None:
    # Nada mudou, evita nova busca
    if searchterm == st.session_state[key]["search"]:
        return

    st.session_state[key]["search"] = searchterm

    ts_start = datetime.datetime.now()

    search_results = search_function(searchterm, **kwargs)

    if search_results is None:
        search_results = []

    st.session_state[key]["options_js"] = _list_to_options_js(search_results)
    st.session_state[key]["options_py"] = _list_to_options_py(search_results)

    if rerun_on_update:
        ts_stop = datetime.datetime.now()
        execution_time_ms = (ts_stop - ts_start).total_seconds() * 1000

        # Aguarda até que o tempo mínimo de execução seja alcançado
        if execution_time_ms < min_execution_time:
            time.sleep((min_execution_time - execution_time_ms) / 1000)

        # Apenas passa o escopo se a versão for >= 1.37
        if st.__version__ >= "1.37":
            rerun(scope=rerun_scope)  # type: ignore
        else:
            rerun()


def _set_defaults(
    key: str,
    default: Any,
    default_options: List[Any] | None = None,
) -> None:
    st.session_state[key] = {
        # Atualizado após cada seleção / reset
        "result": default,
        # Atualizado após cada keystroke de busca
        "search": "",
        # Atualizado após cada execução da search_function
        "options_js": [],
        # Chave usada pelo componente React, usa sufixo de tempo para recarregar após limpar
        "key_react": f"{key}_react_{str(time.time())}",
    }

    if default_options:
        st.session_state[key]["options_js"] = _list_to_options_js(default_options)
        st.session_state[key]["options_py"] = _list_to_options_py(default_options)


ClearStyle = TypedDict(
    "ClearStyle",
    {
        # Determina qual ícone é usado para o botão de limpar
        "icon": Literal["circle-unfilled", "circle-filled", "cross"],
        # Determina quando o botão de limpar é mostrado
        "clearable": Literal["always", "never", "after-submit"],
        # Estilos CSS adicionais para o botão de limpar
        "width": int,
        "height": int,
        "fill": str,
        "stroke": str,
        "stroke-width": int,
    },
    total=False,
)

DropdownStyle = TypedDict(
    "DropdownStyle",
    {
        # Se deve girar o dropdown quando o menu está aberto
        "rotate": bool,
        # Estilos CSS adicionais para o dropdown
        "width": int,
        "height": int,
        "fill": str,
    },
    total=False,
)


class SearchboxStyle(TypedDict, total=False):
    menuList: dict | None
    singleValue: dict | None
    input: dict | None
    placeholder: dict | None
    control: dict | None
    option: dict | None


class StyleOverrides(TypedDict, total=False):
    wrapper: dict | None
    clear: ClearStyle | None
    dropdown: DropdownStyle | None
    searchbox: SearchboxStyle | None


@wrap_inactive_session
def st_searchbox(
    search_function: Callable[[str], List[Any]],
    placeholder: str = "Search ...",
    label: str | None = None,
    default: Any = None,
    default_use_searchterm: bool = False,
    default_options: List[Any] | None = None,
    clear_on_submit: bool = False,
    rerun_on_update: bool = True,
    edit_after_submit: Literal["disabled", "current", "option", "concat"] = "disabled",
    style_absolute: bool = False,
    style_overrides: StyleOverrides | None = None,
    debounce: int = 150,
    min_execution_time: int = MIN_EXECUTION_TIME_DEFAULT,
    reset_function: Callable[[], None] | None = None,
    key: str = "searchbox",
    rerun_scope: Literal["app", "fragment"] = "app",
    reset_interval: int | None = None,  # Novo parâmetro adicionado
    **kwargs,
) -> Any:
    """
    Cria uma nova instância de searchbox, que fornece sugestões com base na entrada do usuário
    e retorna uma opção selecionada ou string vazia se nada for selecionado

    Args:
        search_function (Callable[[str], List[any]]):
            Função que é chamada para buscar novas sugestões após a entrada do usuário.
        placeholder (str, optional):
            Texto mostrado na searchbox quando vazia. Padrão é "Search ...".
        label (str, optional):
            Rótulo mostrado acima da searchbox. Padrão é None.
        default (any, optional):
            Valor retornado se nada foi selecionado até agora. Padrão é None.
        default_use_searchterm (bool, optional):
            Retorna o termo de busca atual se nada foi selecionado. Padrão é False.
        default_options (List[any], optional):
            Lista inicial de opções. Padrão é None.
        clear_on_submit (bool, optional):
            Remove as sugestões ao selecionar. Padrão é False.
        rerun_on_update (bool, optional):
            Reexecuta o app Streamlit após cada busca. Padrão é True.
        edit_after_submit ("disabled", "current", "option", "concat", optional):
            Edita o termo de busca após o envio. Padrão é "disabled".
        style_absolute (bool, optional):
            Posiciona a searchbox de forma absoluta na página. Padrão é False.
        style_overrides (StyleOverrides, optional):
            Estilos CSS passados diretamente para os componentes React. Padrão é None.
        debounce (int, optional):
            Tempo em milissegundos para esperar antes de enviar a entrada para a função de busca.
            Padrão é 150.
        min_execution_time (int, optional):
            Tempo mínimo de execução para a função de busca em milissegundos.
            Padrão é MIN_EXECUTION_TIME_DEFAULT.
        reset_function (Callable[[], None], optional):
            Função chamada após o usuário redefinir a combobox. Padrão é None.
        key (str, optional):
            Chave de sessão do Streamlit. Padrão é "searchbox".
        rerun_scope ("app", "fragment", optional):
            Escopo em que o app Streamlit será reexecutado. Padrão é "app".
        reset_interval (int, optional):
            Intervalo de tempo em segundos após o qual o estado do searchbox será resetado.
            Se None, o estado não será resetado automaticamente. Padrão é None.

    Returns:
        any: baseado na seleção do usuário
    """
    # Ajustar a chave com base no reset_interval
    if reset_interval is not None:
        key_suffix = f"_{int(time.time() // reset_interval)}"
        key_with_interval = key + key_suffix
    else:
        key_with_interval = key

    if key_with_interval not in st.session_state:
        _set_defaults(key_with_interval, default, default_options)

    # Tudo aqui é passado para o React como this.props.args
    react_state = _get_react_component(
        options=st.session_state[key_with_interval]["options_js"],
        clear_on_submit=clear_on_submit,
        placeholder=placeholder,
        label=label,
        edit_after_submit=edit_after_submit,
        style_overrides=style_overrides,
        debounce=debounce,
        # Retorno do estado do React dentro do st.session_state
        key=st.session_state[key_with_interval]["key_react"],
    )

    if style_absolute:
        # Adiciona blocos markdown vazios para reservar espaço para o iframe
        st.markdown("")
        st.markdown("")

        css = """
        iframe[title="streamlit_searchbox.searchbox"] {
            position: absolute;
            z-index: 10;
        }
        """
        st.markdown(f"<style>{css}</style>", unsafe_allow_html=True)

    if react_state is None:
        return st.session_state[key_with_interval]["result"]

    interaction, value = react_state["interaction"], react_state["value"]

    if interaction == "search":
        if default_use_searchterm:
            st.session_state[key_with_interval]["result"] = value

        # Dispara o rerun, nenhuma operação posterior é executada
        _process_search(
            search_function,
            key_with_interval,
            value,
            rerun_on_update,
            rerun_scope=rerun_scope,
            min_execution_time=min_execution_time,
            **kwargs,
        )

    if interaction == "submit":
        st.session_state[key_with_interval]["result"] = (
            st.session_state[key_with_interval]["options_py"][value]
            if "options_py" in st.session_state[key_with_interval]
            else value
        )
        return st.session_state[key_with_interval]["result"]

    if interaction == "reset":
        _set_defaults(key_with_interval, default, default_options)

        if reset_function is not None:
            reset_function()

        if rerun_on_update:
            if st.__version__ >= "1.37":
                rerun(scope=rerun_scope)  # type: ignore
            else:
                rerun()

        return default

    # Nenhuma nova interação do React ocorreu
    return st.session_state[key_with_interval]["result"]

```
