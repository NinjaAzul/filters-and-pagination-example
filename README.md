# filters-and-pagination-example

```js

import React, { useCallback, useEffect, useMemo, useState } from 'react'
import { withAuthorization } from 'hoc/withAuthorization'
import { useNavbar } from 'contexts/navbar'
import { v4 as uuid } from 'uuid'
import {
  Container,
  Content,
  Wrapper,
  FilterContainer,
  Title,
  DropdownContainer,
  CustomDropdown,
  Line,
  Email,
  Name,
  Status,
  ErrorContainer,
  ErrorWrapper,
  ErrorContent,
  ErrorIcon,
  ErrorIconContainer,
  ErrorKey,
  ErrorValue,
  ErrorTitle,
  Errors,
  LineError,
  LineCollorBLue,
} from './style'
import { showToaster } from 'containers/common/Toast'
import Header from 'components/ImportLeadsComponents/Header'
import { NoneFound, Icons, Text, Button, Table } from '@randon/design-system'
import { withTheme } from 'styled-components'
import ImportLeadModal from 'components/ImportLeadsComponents/ImportLeadModal'
import { getDocumentToImportLeads, uploadFileToImportLeads, isGeneralAdmin } from 'services'
import SearchField from 'components/SearchField'

const options = [
  { id: 'ALL', label: 'Todos' },
  { id: 'VALIDATION_ERROR', label: 'Erro na validação' },
  { id: 'LEAD_EXISTS', label: 'Lead já existe' },
  { id: 'SUCCESS', label: 'Sucesso' },
  { id: 'DATABASE_ERROR', label: 'Erro inesperado' },
  { id: 'LIMIT_EXCEEDED', label: 'Limite excedido' },
]

const LeadsImport = ({ theme }) => {
  const { setIsVisible } = useNavbar()
  const [isImportModalOpen, setIsImportModalOpen] = useState(false)
  const [resultImportLeads, setResultImportLeads] = useState([])
  const [loadingImportLeads, setLoadingImportLeads] = useState(false)
  const [pageCount, setPageCount] = useState(0)

  const [filters, setFiltersState] = useState({
    state: 'ALL',
    search: '',
    page: 1,
    limit: 10,
  })

  const applyFilters = useCallback(
    filter => setFiltersState(oldFilters => ({ ...oldFilters, ...filter })),
    [setFiltersState],
  )

  const setDebouncedSearch = useCallback(search => applyFilters({ search, page: 1 }), [
    applyFilters,
  ])

  const setSelectedStatesSearch = useCallback(state => applyFilters({ state, page: 1 }), [
    applyFilters,
  ])

  useEffect(() => {
    setIsVisible(false)

    return () => setIsVisible(true)
  }, [setIsVisible])

  const onDownload = async values => {
    try {
      await getDocumentToImportLeads(values.salePointId)
      showToaster({
        theme,
        success: true,
        title: 'Documento gerado com sucesso!',
      })
    } catch (error) {
      console.error(error)
      showToaster({
        theme,
        success: false,
        title: error?.response?.data?.message || 'Algo deu errado, tente novamente mais tarde!',
      })
    }
  }

  const onUpload = async values => {
    try {
      setLoadingImportLeads(true)
      const [{ file }] = values.spreadsheet
      const { data } = await uploadFileToImportLeads(file)
      setResultImportLeads(data.map(result => ({ ...result, id: uuid() })))
      showToaster({
        theme,
        success: true,
        title: 'Documento enviado com sucesso!',
      })
    } catch (error) {
      console.error(error)
      showToaster({
        theme,
        success: false,
        title: error?.response?.data?.message || 'Algo deu errado, tente novamente mais tarde!',
      })
    } finally {
      setLoadingImportLeads(false)
    }
  }

  const filteredList = useMemo(() => {
    const filtered = resultImportLeads
      .filter(item => {
        if (filters.state === 'ALL') return true
        return item.type === filters.state
      })
      .filter(item => {
        if (!filters.search) return true
        return (
          item.rowData.email.toLowerCase().includes(filters.search.toLowerCase()) ||
          item.rowData.name.toLowerCase().includes(filters.search.toLowerCase())
        )
      })

    setPageCount(Math.ceil(filtered.length / filters.limit))

    return filtered.slice((filters.page - 1) * filters.limit, filters.page * filters.limit)
  }, [resultImportLeads, filters])

  const TYPE = {
    LEAD_EXISTS: 'Lead já existe',
    VALIDATION_ERROR: 'Erro na validação',
    SUCCESS: 'Sucesso',
    DATABASE_ERROR: 'Erro inesperado',
    LIMIT_EXCEEDED: 'Limite excedido',
  }

  const headers = [
    {
      name: 'LINHA',
      getValue: l => <Line>{`#${l.rowIndex}`}</Line>,
    },

    {
      name: 'NOME',
      getValue: l => <Name>{l?.rowData?.name || ''}</Name>,
    },
    {
      name: 'E-MAIL',
      getValue: l => <Email>{l?.rowData?.email || ''}</Email>,
    },
    {
      name: 'STATUS',
      getValue: l => <Status status={l.type}>{TYPE[l.type]}</Status>,
    },
  ]

  const mobileHeaders = [
    {
      name: 'LINHA',
      getValue: l => <Line>{`#${l.rowIndex}`}</Line>,
    },

    {
      name: 'NOME',
      getValue: l => <span>{l?.rowData?.name || ''}</span>,
    },
    {
      name: 'E-MAIL',
      getValue: l => <span>{l?.rowData?.email || ''}</span>,
    },
    {
      name: 'STATUS',
      getValue: l => <span>{l.type}</span>,
    },
  ]

  const headersChildren = [
    {
      getValue: l => {
        return (
          <ErrorContainer>
            <ErrorWrapper>
              <ErrorIconContainer>
                <ErrorIcon />
              </ErrorIconContainer>
              <ErrorContent>
                <LineError>{`Linha #${l.errors[0].rowIndex}`}</LineError>
                <ErrorTitle>Lista de erros:</ErrorTitle>
                {l.errors.map(error => {
                  const { rowIndex, ...rest } = error

                  return (
                    <ErrorWrapper>
                      {Object.keys(rest).length > 0 && <LineCollorBLue />}
                      <Errors key={rowIndex}>
                        {Object.entries(rest).map(([key, value]) => {
                          return (
                            <ErrorWrapper key={`${key}-${error.rowIndex}`}>
                              <ErrorKey>Mensagem:</ErrorKey>
                              <ErrorValue>{value}.</ErrorValue>
                            </ErrorWrapper>
                          )
                        })}
                      </Errors>
                    </ErrorWrapper>
                  )
                })}
              </ErrorContent>
            </ErrorWrapper>
          </ErrorContainer>
        )
      },
    },
  ]

  const headersChildrenMobile = [
    {
      getValue: l => (
        <span role="img" aria-label="building mobile">
          🚧🚧
        </span>
      ),
    },
  ]

  return (
    <>
      <Container>
        <Header />

        <Content>
          <Wrapper>
            <Title>Relatório de importação</Title>

            <FilterContainer>
              <SearchField
                placeholder="Pesquisar..."
                setSearch={setDebouncedSearch}
                debounceDelay={500}
              />
              <DropdownContainer>
                <Text color={theme.colors.Grey3}>Filtros:</Text>
                <CustomDropdown
                  value={filters.state}
                  onChange={setSelectedStatesSearch}
                  withoutHighlight
                  placeholder="Status"
                  options={options}
                />
              </DropdownContainer>
              <Button onClick={() => setIsImportModalOpen(true)}>Importar leads</Button>
            </FilterContainer>
          </Wrapper>
          {resultImportLeads.length > 0 && (
            <Text
              variant={Text.variants.LARGE}
            >{`${resultImportLeads.length} leads processados`}</Text>
          )}

          {loadingImportLeads && <Table.Loader rows={5} />}

          {!loadingImportLeads && (
            <Table
              headers={headers}
              mobileHeaders={mobileHeaders}
              data={filteredList}
              whenEmpty={
                <NoneFound
                  icon={<Icons.EmptystateSearchproducts />}
                  text="Nenhum relatório encontrado."
                  description="Faça o upload da planilha preenchida para a importação dos dados."
                />
              }
            >
              {({ data }) => (
                <Table
                  striped
                  noHeaders
                  headers={headersChildren}
                  mobileHeaders={headersChildrenMobile}
                  data={[
                    {
                      errors: data.messages.map(message => ({
                        ...message,
                        rowIndex: data.rowIndex,
                      })),
                    },
                  ]}
                />
              )}
            </Table>
          )}

          <Table.Pagination
            currentPage={filters.page}
            onPageChange={p => applyFilters({ page: p })}
            pageCount={pageCount}
          />
        </Content>
      </Container>

      <ImportLeadModal
        isOpen={isImportModalOpen}
        onClose={() => setIsImportModalOpen(false)}
        onDownload={onDownload}
        onUpload={onUpload}
        isLoading={false}
      />
    </>
  )
}

export default withAuthorization(async () => {
  const { data } = await isGeneralAdmin()

  const isGeneralAdminResponse = data.data
  return isGeneralAdminResponse
}, withTheme(LeadsImport))


```
