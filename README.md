

from itertools import chain
from typing import Annotated

from aiohttp import ClientSession
from fastapi import APIRouter, Depends, HTTPException, status
from redis.asyncio.client import Redis
import logging

from plexio import __version__
from plexio.dependencies import (
    get_addon_configuration,
    get_cache,
    get_http_client,
    set_sentry_user,
)
from plexio.models import PLEX_TO_STREMIO_MEDIA_TYPE, STREMIO_TO_PLEX_MEDIA_TYPE
from plexio.models.addon import AddonConfiguration
from plexio.models.stremio import (
    StremioCatalog,
    StremioCatalogManifest,
    StremioManifest,
    StremioMediaType,
    StremioMetaResponse,
    StremioStreamsResponse,
)
from plexio.plex.media_server_api import (
    get_all_episodes,
    get_media,
    get_section_media,
    stremio_to_plex_id,
)

router = APIRouter()
router.dependencies.append(Depends(set_sentry_user))

# Initialize logger
logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO)

@router.get('/manifest.json', response_model_exclude_none=True)
@router.get(
    '/{installation_id}/{base64_cfg}/manifest.json', response_model_exclude_none=True
)
async def get_manifest(
    configuration: Annotated[AddonConfiguration | None, Depends(get_addon_configuration)],
    installation_id: str | None = None,
) -> StremioManifest:
    """
    Retrieve the manifest for the Stremio plugin, based on the configuration.
    """
    catalogs = []
    description = 'Play movies and series from plex.tv.'
    name = 'Plexio'

    if configuration is not None:
        for section in configuration.sections:
            catalogs.append(
                StremioCatalogManifest(
                    id=section.key,
                    type=PLEX_TO_STREMIO_MEDIA_TYPE[section.type],
                    name=f'{section.title} | {configuration.server_name}',
                    extra=[
                        {'name': 'skip', 'isRequired': False},
                        {'name': 'search', 'isRequired': False},
                    ],
                ),
            )

        name += f' ({configuration.server_name})'
        description += f' Your installation ID: {installation_id}'

    logger.info(f"Returning manifest with {len(catalogs)} catalogs.")
    return StremioManifest(
        id='com.stremio.plexio',
        version=__version__,
        description=description,
        name=name,
        resources=[
            'stream',
            'catalog',
            {
                'name': 'meta',
                'types': ['movie', 'series'],
                'idPrefixes': ['plex://', 'local://'],
            },
        ],
        types=[StremioMediaType.movie, StremioMediaType.series],
        catalogs=catalogs,
        idPrefixes=['tt', 'plex://', 'local://'],
        behaviorHints={
            'configurable': True,
            'configurationRequired': configuration is None,
        },
        contactEmail='support@plexio.stream',
    )

@router.get(
    '/{installation_id}/{base64_cfg}/catalog/{stremio_type}/{catalog_id}.json',
    response_model_exclude_none=True,
)
@router.get(
    '/{installation_id}/{base64_cfg}/catalog/{stremio_type}/{catalog_id}/skip={skip}.json',
    response_model_exclude_none=True,
)
@router.get(
    '/{installation_id}/{base64_cfg}/catalog/{stremio_type}/{catalog_id}/search={search}.json',
    response_model_exclude_none=True,
)
async def get_catalog(
    http: Annotated[ClientSession, Depends(get_http_client)],
    configuration: Annotated[AddonConfiguration, Depends(get_addon_configuration)],
    stremio_type: StremioMediaType,
    catalog_id: str,
    skip: int = 0,
    search: str = '',
) -> StremioCatalog:
    """
    Fetch a catalog of media from Plex based on the section id and other parameters.
    """
    try:
        media = await get_section_media(
            client=http,
            url=configuration.discovery_url,
            token=configuration.access_token,
            section_id=catalog_id,
            search=search,
            skip=skip,
        )
        logger.info(f"Fetched {len(media)} items from catalog {catalog_id}.")
        return StremioCatalog(
            metas=[m.to_stremio_meta_review(configuration) for m in media],
        )
    except Exception as e:
        logger.error(f"Error fetching catalog {catalog_id}: {str(e)}")
        raise HTTPException(status_code=status.HTTP_500_INTERNAL_SERVER_ERROR, detail="Error fetching catalog data.")

@router.get(
    '/{installation_id}/{base64_cfg}/meta/{stremio_type}/{plex_id:path}.json',
    response_model_exclude_none=True,
)
async def get_meta(
    http: Annotated[ClientSession, Depends(get_http_client)],
    configuration: Annotated[AddonConfiguration, Depends(get_addon_configuration)],
    stremio_type: StremioMediaType,
    plex_id: str,
) -> StremioMetaResponse:
    """
    Retrieve metadata for a given Plex ID.
    """
    try:
        media = await get_media(
            client=http,
            url=configuration.discovery_url,
            token=configuration.access_token,
            guid=plex_id,
            get_only_first=True,
        )
        if not media:
            raise HTTPException(status_code=status.HTTP_404_NOT_FOUND)
        
        media = media[0]
        meta = media.to_stremio_meta(configuration)

        if stremio_type == StremioMediaType.series:
            episodes = await get_all_episodes(
                client=http,
                url=configuration.discovery_url,
                token=configuration.access_token,
                key=media.key,
            )
            meta.videos = [e.to_stremio_video_meta(configuration) for e in episodes]

        return StremioMetaResponse(meta=meta)
    except Exception as e:
        logger.error(f"Error fetching metadata for {plex_id}: {str(e)}")
        raise HTTPException(status_code=status.HTTP_500_INTERNAL_SERVER_ERROR, detail="Error fetching metadata.")

@router.get(
    '/{installation_id}/{base64_cfg}/stream/{stremio_type}/{media_id:path}.json',
    response_model_exclude_none=True,
)
async def get_stream(
    http: Annotated[ClientSession, Depends(get_http_client)],
    cache: Annotated[Redis, Depends(get_cache)],
    configuration: Annotated[AddonConfiguration, Depends(get_addon_configuration)],
    stremio_type: StremioMediaType,
    media_id: str,
) -> StremioStreamsResponse:
    """
    Get stream information for a media item, converting Stremio ID to Plex ID if necessary.
    """
    try:
        if media_id.startswith('tt'):
            plex_id = await stremio_to_plex_id(
                client=http,
                url=configuration.discovery_url,
                token=configuration.access_token,
                cache=cache,
                stremio_id=media_id,
                media_type=STREMIO_TO_PLEX_MEDIA_TYPE[stremio_type],
            )
            if not plex_id:
                return StremioStreamsResponse()
        else:
            plex_id = media_id

        media = await get_media(
            client=http,
            url=configuration.discovery_url,
            token=configuration.access_token,
            guid=plex_id,
        )
        return StremioStreamsResponse(
            streams=chain.from_iterable(
                meta.get_stremio_streams(configuration) for meta in media
            ),
        )
    except Exception as e:
        logger.error(f"Error fetching stream for {media_id}: {str(e)}")
        raise HTTPException(status_code=status.HTTP_500_INTERNAL_SERVER_ERROR, detail="Error fetching stream data.")
