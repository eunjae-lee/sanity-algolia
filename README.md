# Sanity + Algolia = ♥️

Here are some helpers to facilitate indexing your Sanity documents as Algolia records, via custom serializing, optional hidden/visibility filtering and directly syncing to an Algolia index from a Sanity webhook.

## Webhook example

This is an example of indexing Sanity content in Algolia directly from a [Sanity webhook](https://www.sanity.io/docs/webhooks). The target of the webhook is a serverless function running on Vercel, Netlify, AWS etc. You can configure Sanity webhooks at [manage.sanity.io](https://manage.sanity.io) or via the `sanity hook` command from your Sanity Studio folder. You should configure your webhook to target the URL of the serverless function once deployed.

### Installing

```
npm i sanity-algolia
```

### Use in your serverless function

Note that your serverless hosting might require a build step to properly deploy your serverless functions, and that the exported handler and passed parameters might differ from the following example, which is TypeScript on Vercel. Please refer to documentation on deploying functions at your hosting service of choice in order to adapt it to your own needs.

```typescript
import algoliasearch from 'algoliasearch'
import sanityClient, { SanityDocumentStub } from '@sanity/client'
import { NowRequest, NowResponse } from '@vercel/node'
import indexer, { flattenBlocks } from 'sanity-algolia'

const algolia = algoliasearch('application-id', 'api-key')
const sanity = sanityClient({
  projectId: 'my-sanity-project-id',
  dataset: 'my-dataset-name',
  // If your dataset is private you need to add a read token.
  // You can mint one at https://manage.sanity.io
  token: 'read-token',
  useCdn: false,
})

/**
 *  This function receives webhook POSTs from Sanity and updates, creates or
 *  deletes records in the corresponding Algolia indices.
 */
const handler = (req: NowRequest, res: NowResponse) => {
  // Tip: Its good practice to include a shared secret in your webhook URLs and
  // validate it before proceeding with webhook handling. Omitted in this short
  // example.
  if (req.headers['content-type'] !== 'application/json') {
    res.status(400)
    res.json({ message: 'Bad request' })
    return
  }

  // Configure this to match an existing Algolia index name
  const algoliaIndex = algolia.initIndex('my-index')

  const sanityAlgolia = indexer(
    // The first parameter maps a Sanity document type to its respective Algolia
    // search index. In this example both `post` and `article` Sanity types live
    // in the same Algolia index. Optionally you can also customize how the
    // document is fetched from Sanity by specifying a GROQ projection.
    {
      post: { index: algoliaIndex },
      // For the article document in this example we want to resolve a list of
      // references to authors. We can do this by customizing the projection for
      // the article type. Here we fetch heading, body and a resolved array of
      // author documents.
      article: {
        index: algoliaIndex,
        projection: '{heading, body, authors[]->}',
      },
    },

    // The second parameter is a function that maps from a fetched Sanity document
    // to an Algolia Record. Notice the flattenBlocks method used for extracting the
    // raw string values from portable text in this example.
    (document: SanityDocumentStub) => {
      switch (document._type) {
        case 'post':
          return {
            title: document.title,
            path: document.slug.current,
            body: flattenBlocks(document.body),
          }
        case 'article':
          return {
            title: document.heading,
            body: flattenBlocks(document.body),
            authorNames: document.authors.map((a) => a.name),
          }
        default:
          throw new Error('You didnt handle a type you declared interest in')
      }
    },
    // Visibility function (optional).
    //
    // The third parameter is an optional visibility function. Returning `true`
    // for a given document here specifies that it should be indexed for search
    // in Algolia. This is handy if for instance a field value on the document
    // decides if it should be indexed or not. This would also be the place to
    // implement any `publishedAt` datetime visibility rules or other custom
    // visibility scheme you may be using.
    (document: SanityDocumentStub) => {
      if (document.hasOwnProperty('isHidden')) {
        return !document.isHidden
      }
      return true
    }
  )

  // Finally connect the Sanity webhook payload to Algolia indices via the
  // configured serializers and optional visibility function. `webhookSync` will
  // inspect the webhook payload, make queries back to Sanity with the `sanity`
  // client and make sure the algolia indices are synced to match.
  return sanityAlgolia
    .webhookSync(sanity, req.body)
    .then(() => res.status(200).send('ok'))
}

export default handler
```

## Todos

- [x] Easily customize projections for resolving references
- [ ] Use Algolia batch APIs?
- [ ] Example of initial indexing of existing content
- [ ] Handle situations where the record is too large to index.

## Links

- [Sanity webhook documentataion](https://www.sanity.io/docs/webhooks)
- [Algolia indexing documentation](https://www.algolia.com/doc/api-client/methods/indexing/)
- [The GROQ Query language](https://www.sanity.io/docs/groq)
- [Vercel Serverless Functions documentation](https://vercel.com/docs/serverless-functions/introduction)
- [Netlify functions documentation](https://docs.netlify.com/functions/build-with-javascript/)
